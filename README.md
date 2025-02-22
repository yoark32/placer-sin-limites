// Plataforma estilo Pornhub con monetización, pagos y modo premium con comisión para el dueño

const express = require('express');
const mongoose = require('mongoose');
const multer = require('multer');
const stripe = require('stripe')('TU_CLAVE_STRIPE');
const paypal = require('@paypal/checkout-server-sdk');
const cors = require('cors');
const jwt = require('jsonwebtoken');
const bcrypt = require('bcryptjs');
require('dotenv').config();

const app = express();
app.use(express.json());
app.use(cors());

mongoose.connect(process.env.MONGO_URI, {
    useNewUrlParser: true,
    useUnifiedTopology: true,
});

// Esquemas de usuarios y videos
const UserSchema = new mongoose.Schema({
    username: String,
    email: String,
    password: String,
    isPremium: { type: Boolean, default: false },
    balance: { type: Number, default: 0 },
    earnings: { type: Number, default: 0 },
});

const VideoSchema = new mongoose.Schema({
    title: String,
    description: String,
    url: String,
    owner: mongoose.Schema.Types.ObjectId,
    monetized: { type: Boolean, default: false },
    earnings: { type: Number, default: 0 },
    views: { type: Number, default: 0 },
});

const User = mongoose.model('User', UserSchema);
const Video = mongoose.model('Video', VideoSchema);

const COMMISSION_RATE = 0.30; // 30% de comisión para el dueño de la plataforma

// Autenticación
app.post('/register', async (req, res) => {
    const { username, email, password } = req.body;
    const hashedPassword = await bcrypt.hash(password, 10);
    const newUser = new User({ username, email, password: hashedPassword });
    await newUser.save();
    res.json({ message: 'Usuario registrado con éxito' });
});

app.post('/login', async (req, res) => {
    const { email, password } = req.body;
    const user = await User.findOne({ email });
    if (!user || !(await bcrypt.compare(password, user.password))) {
        return res.status(401).json({ error: 'Credenciales incorrectas' });
    }
    const token = jwt.sign({ userId: user._id }, 'secreto', { expiresIn: '1h' });
    res.json({ token, isPremium: user.isPremium });
});

// Subida de videos (integrar con AWS S3 o Firebase)
const storage = multer.memoryStorage();
const upload = multer({ storage: storage });

app.post('/upload', upload.single('video'), async (req, res) => {
    res.json({ message: 'Video subido correctamente' });
});

// Pago de suscripción premium
app.post('/pay-premium', async (req, res) => {
    const { userId, paymentMethod } = req.body;
    if (paymentMethod === 'paypal') {
        // Configurar pago con PayPal
    } else if (paymentMethod === 'stripe') {
        const paymentIntent = await stripe.paymentIntents.create({
            amount: 999,
            currency: 'usd',
        });
        res.json({ clientSecret: paymentIntent.client_secret });
    }
});

// Monetización y distribución de ganancias
app.post('/monetize', async (req, res) => {
    const { userId, amount } = req.body;
    const user = await User.findById(userId);
    if (!user) return res.status(404).json({ error: 'Usuario no encontrado' });
    
    const ownerShare = amount * COMMISSION_RATE;
    const userShare = amount - ownerShare;
    user.earnings += userShare;
    await user.save();
    
    res.json({ message: 'Ganancias distribuidas', ownerShare, userShare });
});

app.listen(5000, () => console.log('Servidor corriendo en el puerto 5000'));

// Frontend: React para subir videos y monetizar con comisión
import React, { useState } from 'react';
import axios from 'axios';

const UploadVideo = () => {
    const [file, setFile] = useState(null);
    const handleUpload = async () => {
        const formData = new FormData();
        formData.append('video', file);
        await axios.post('http://localhost:5000/upload', formData);
        alert('Video subido con éxito');
    };

    return (
        <div>
            <h2>Sube tu video y gana dinero</h2>
            <input type="file" onChange={(e) => setFile(e.target.files[0])} />
            <button onClick={handleUpload}>Subir</button>
        </div>
    );
};

export default UploadVideo;
