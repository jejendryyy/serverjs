# serverjs
// backend/server.js

const express = require('express');
const cors = require('cors');
const multer = require('multer');
const path = require('path');
const fs = require('fs');

const app = express();
const PORT = 5000;

// Middleware
app.use(cors());
app.use(express.json());
// Menyajikan file statis dari folder 'uploads'
app.use('/uploads', express.static(path.join(__dirname, 'uploads')));

// --- Konfigurasi Multer untuk Upload Foto ---
const storage = multer.diskStorage({
  destination: function (req, file, cb) {
    const uploadPath = 'uploads/';
    // Buat direktori jika belum ada
    if (!fs.existsSync(uploadPath)) {
      fs.mkdirSync(uploadPath);
    }
    cb(null, uploadPath);
  },
  filename: function (req, file, cb) {
    // Buat nama file yang unik untuk menghindari konflik
    cb(null, Date.now() + path.extname(file.originalname));
  }
});

const upload = multer({ storage: storage });

// --- Database In-Memory Sederhana ---
let siswaDb = [
  { id: 1, nama: "Budi Santoso", kelas: "12A", alamat: "Jl. Merdeka No. 1", foto: "default.png" },
  { id: 2, nama: "Ani Lestari", kelas: "11B", alamat: "Jl. Pahlawan No. 22", foto: "default.png" }
];
let lastId = 2;

// --- Routes API (CRUD) ---

// READ (Get All Siswa)
app.get('/api/siswa', (req, res) => {
  res.json(siswaDb);
});

// CREATE (Add New Siswa) - dengan upload foto
app.post('/api/siswa', upload.single('foto'), (req, res) => {
  const { nama, kelas, alamat } = req.body;
  const newSiswa = {
    id: ++lastId,
    nama,
    kelas,
    alamat,
    // Jika ada file yang diupload, gunakan namanya, jika tidak, gunakan default
    foto: req.file ? req.file.filename : 'default.png'
  };
  siswaDb.push(newSiswa);
  res.status(201).json(newSiswa);
});

// UPDATE (Edit Siswa)
app.put('/api/siswa/:id', upload.single('foto'), (req, res) => {
  const { id } = req.params;
  const { nama, kelas, alamat } = req.body;
  const siswaIndex = siswaDb.findIndex(s => s.id == id);

  if (siswaIndex === -1) {
    return res.status(404).json({ message: 'Siswa tidak ditemukan' });
  }

  const siswaToUpdate = siswaDb[siswaIndex];
  siswaToUpdate.nama = nama || siswaToUpdate.nama;
  siswaToUpdate.kelas = kelas || siswaToUpdate.kelas;
  siswaToUpdate.alamat = alamat || siswaToUpdate.alamat;
  
  if (req.file) {
    // Hapus foto lama jika bukan default
    if (siswaToUpdate.foto && siswaToUpdate.foto !== 'default.png') {
        const oldPath = path.join(__dirname, 'uploads', siswaToUpdate.foto);
        if (fs.existsSync(oldPath)) {
            fs.unlinkSync(oldPath);
        }
    }
    siswaToUpdate.foto = req.file.filename;
  }

  siswaDb[siswaIndex] = siswaToUpdate;
  res.json(siswaToUpdate);
});

// DELETE (Remove Siswa)
app.delete('/api/siswa/:id', (req, res) => {
  const { id } = req.params;
  const siswaIndex = siswaDb.findIndex(s => s.id == id);

  if (siswaIndex === -1) {
    return res.status(404).json({ message: 'Siswa tidak ditemukan' });
  }

  // Hapus foto terkait jika bukan default
  const siswaToDelete = siswaDb[siswaIndex];
  if (siswaToDelete.foto && siswaToDelete.foto !== 'default.png') {
    const filePath = path.join(__dirname, 'uploads', siswaToDelete.foto);
     if (fs.existsSync(filePath)) {
        fs.unlinkSync(filePath);
     }
  }

  siswaDb = siswaDb.filter(s => s.id != id);
  res.status(204).send(); // No Content
});


// Jalankan Server
app.listen(PORT, () => {
  console.log(🚀 Server berjalan di http://localhost:${PORT});
});
