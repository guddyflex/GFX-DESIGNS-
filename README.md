const express = require("express");
const mongoose = require("mongoose");
const cors = require("cors");
const jwt = require("jsonwebtoken");
const bcrypt = require("bcryptjs");
const axios = require("axios");
require("dotenv").config();

const app = express();
app.use(cors());
app.use(express.json());
app.use(express.static("public"));

mongoose.connect(process.env.MONGO_URI)
.then(() => console.log("DB Connected"));

// ================= MODELS =================
const User = mongoose.model("User", {
  name: String,
  email: String,
  password: String,
  role: String
});

const Gig = mongoose.model("Gig", {
  title: String,
  description: String,
  price: Number,
  file: String,
  sellerEmail: String
});

const Order = mongoose.model("Order", {
  gigId: String,
  buyerEmail: String,
  amount: Number,
  reference: String,
  status: String,
  sellerEmail: String,
  adminFee: Number
});

const Wallet = mongoose.model("Wallet", {
  email: String,
  balance: Number
});

// ================= ADMIN FEE =================
let PLATFORM_FEE = 0.15;

// ================= CREATE GIG =================
app.post("/gig", async (req, res) => {
  const gig = new Gig(req.body);
  await gig.save();
  res.json(gig);
});

// ================= GET GIGS =================
app.get("/gigs", async (req, res) => {
  res.json(await Gig.find());
});

// ================= PAY =================
app.post("/pay", async (req, res) => {
  const { email, amount, gigId } = req.body;

  const response = await axios.post("https://api.paystack.co/transaction/initialize", {
    email,
    amount: amount * 100,
    callback_url: `${process.env.BASE_URL}/verify?gigId=${gigId}`
  }, {
    headers: {
      Authorization: `Bearer ${process.env.PAYSTACK_SECRET}`
    }
  });

  res.json(response.data);
});

// ================= VERIFY =================
app.get("/verify", async (req, res) => {
  const { reference, gigId } = req.query;

  const response = await axios.get(
    `https://api.paystack.co/transaction/verify/${reference}`,
    {
      headers: {
        Authorization: `Bearer ${process.env.PAYSTACK_SECRET}`
      }
    }
  );

  const data = response.data.data;

  if (data.status === "success") {

    const gig = await Gig.findById(gigId);

    const adminFee = gig.price * PLATFORM_FEE;
    const sellerEarn = gig.price - adminFee;

    await new Order({
      gigId,
      buyerEmail: data.customer.email,
      amount: gig.price,
      reference,
      status: "paid",
      sellerEmail: gig.sellerEmail,
      adminFee
    }).save();

    let wallet = await Wallet.findOne({ email: gig.sellerEmail });

    if (!wallet) wallet = new Wallet({ email: gig.sellerEmail, balance: 0 });

    wallet.balance += sellerEarn;
    await wallet.save();

    return res.send("🎉 Payment successful. Gig unlocked.");
  }

  res.send("❌ Payment failed");
});

// ================= ADMIN =================
app.post("/set-fee", (req, res) => {
  PLATFORM_FEE = req.body.fee;
  res.json({ fee: PLATFORM_FEE });
});

app.listen(process.env.PORT || 3000, "0.0.0.0", () => {
  console.log("Fiverr system running");
});