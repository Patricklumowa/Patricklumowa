# Script Video Presentasi: Compily - Online Rust Compiler

## Durasi Estimasi: 10-15 menit

---

## PEMBUKAAN (Presenter 1)

**[SCENE: Semua presenter di frame]**

**Presenter 1:**
> "Halo semuanya! Selamat datang di presentasi project kami yang berjudul **Compily - Online Rust Compiler**."
>
> "Perkenalkan, saya [Nama] dan bersama saya ada:"
> - "[Nama Presenter 2]"
> - "[Nama Presenter 3]"  
> - "[Nama Presenter 4]"
>
> "Hari ini kami akan mempresentasikan aplikasi web yang memungkinkan pengguna untuk menulis, mengkompilasi, dan menjalankan kode Rust langsung di browser."
>
> "Project ini menggunakan arsitektur full-stack dengan **Backend Rust** menggunakan framework Axum, dan **Frontend React** dengan Vite."

---

## BAGIAN 1: OVERVIEW ARSITEKTUR (Presenter 1)

**[SCENE: Tampilkan diagram arsitektur atau screen share struktur folder]**

**Presenter 1:**
> "Sebelum masuk ke detail, mari kita lihat struktur project kami."
>
> "Project ini terbagi menjadi dua bagian utama:"
>
> **Backend** (folder `backend/`):
> - Ditulis dalam bahasa Rust
> - Menggunakan framework Axum untuk HTTP server
> - Database SQLite untuk menyimpan data user dan snippet
> - Autentikasi menggunakan JWT dan password hashing dengan Argon2
>
> **Frontend** (folder `frontend/`):
> - React 19 dengan Vite sebagai build tool
> - HeroUI dan Tailwind CSS untuk styling
> - Monaco Editor untuk code editor
> - Komunikasi real-time via WebSocket

---

## BAGIAN 2: BACKEND - DATABASE & STATE (Presenter 2)

**[SCENE: Buka file `backend/src/db.rs`]**

**Presenter 2:**
> "Sekarang saya akan menjelaskan bagian backend, dimulai dari database."
>
> "Buka file `db.rs`. Di sini kita mendefinisikan **AppState** yang berisi koneksi database pool."

📍 **`backend/src/db.rs` - Line 4**
```rust
#[derive(Clone)]
pub struct AppState {
    pub db: Pool<Sqlite>,
}
```

> "Fungsi `init_db` bertugas untuk:"
> 1. Membaca `DATABASE_URL` dari environment variable
> 2. Membuat file database jika belum ada
> 3. Membuat tabel `users` dan `snippets`

📍 **`backend/src/db.rs` - Line 24**
```rust
CREATE TABLE IF NOT EXISTS users (
    id TEXT PRIMARY KEY,
    username TEXT UNIQUE NOT NULL,
    password_hash TEXT NOT NULL
);
```

> "Tabel users menyimpan ID, username yang unik, dan password yang sudah di-hash."

📍 **`backend/src/db.rs` - Line 35**
```rust
CREATE TABLE IF NOT EXISTS snippets (
    id TEXT PRIMARY KEY,
    user_id TEXT NOT NULL,
    title TEXT NOT NULL,
    code TEXT NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id)
);
```

> "Tabel snippets menyimpan kode yang disimpan user, dengan foreign key ke tabel users."

---

## BAGIAN 3: BACKEND - AUTENTIKASI (Presenter 2)

**[SCENE: Buka file `backend/src/auth.rs`]**

**Presenter 2:**
> "Selanjutnya, file `auth.rs` menangani register dan login."

📍 **`backend/src/auth.rs` - Line 20**
```rust
pub struct RegisterRequest {
    pub username: String,
    pub password: String,
}
```

> "Kita menerima username dan password dari user."

📍 **`backend/src/auth.rs` - Line 102**
```rust
let salt = SaltString::generate(&mut OsRng);
let argon2 = Argon2::default();
let password_hash = argon2
    .hash_password(payload.password.as_bytes(), &salt)
    .to_string();
```

> "Password **tidak pernah disimpan dalam bentuk plain text**. Kami menggunakan algoritma Argon2 untuk hashing, yang merupakan standar industri untuk keamanan password."

📍 **`backend/src/auth.rs` - Line 156**
```rust
let claims = Claims {
    sub: user_id,
    exp: expiration as usize,
};
let token = encode(&Header::default(), &claims, &EncodingKey::from_secret(secret.as_bytes()));
```

> "Setelah login berhasil, kita generate JWT token yang berlaku 24 jam. Token ini berisi user ID dan waktu expired."

📍 **`backend/src/auth.rs` - Line 48**
```rust
impl<S> FromRequestParts<S> for Claims {
    // ... extract token from Authorization header
    let token = &auth_header[7..]; // Remove "Bearer "
    let token_data = decode::<Claims>(token, &DecodingKey::from_secret(...));
}
```

> "Yang menarik, Axum memungkinkan kita membuat custom extractor. `Claims` bisa langsung digunakan sebagai parameter fungsi, dan Axum otomatis memvalidasi JWT dari header Authorization."

---

## BAGIAN 4: BACKEND - SNIPPET MANAGEMENT (Presenter 3)

**[SCENE: Buka file `backend/src/snippets.rs`]**

**Presenter 3:**
> "Saya akan menjelaskan manajemen snippet. File ini mengimplementasikan full CRUD operations."

📍 **`backend/src/snippets.rs` - Line 56**
```rust
pub async fn create_snippet(
    State(state): State<AppState>,
    claims: Claims,  // <-- JWT sudah divalidasi di sini
    Json(payload): Json<CreateSnippetRequest>,
) -> Result<impl IntoResponse, (StatusCode, String)> {
    let id = uuid::Uuid::new_v4().to_string();
    sqlx::query("INSERT INTO snippets (id, user_id, title, code) VALUES (?, ?, ?, ?)")
        .bind(&id)
        .bind(&claims.sub)  // user_id dari JWT
        .bind(&payload.title)
        .bind(&payload.code)
        .execute(&state.db)
        .await
}
```

> "Perhatikan parameter `claims: Claims`. Ini berarti endpoint ini **membutuhkan autentikasi**. User ID diambil dari JWT, bukan dari request body, sehingga user tidak bisa menyimpan snippet atas nama orang lain."

📍 **`backend/src/snippets.rs` - Line 117**
```rust
sqlx::query_as::<_, Snippet>("SELECT * FROM snippets WHERE id = ? AND user_id = ?")
    .bind(&id)
    .bind(&claims.sub)  // memastikan hanya pemilik yang bisa akses
```

> "Setiap query snippet selalu menyertakan `user_id` untuk memastikan user hanya bisa mengakses snippet miliknya sendiri."

> "Kami juga mengimplementasikan PATCH untuk partial update - user bisa update title saja atau code saja, tidak harus keduanya."

---

## BAGIAN 5: BACKEND - COMPILER ENGINE (Presenter 3)

**[SCENE: Buka file `backend/src/main.rs`]**

**Presenter 3:**
> "Ini adalah inti dari aplikasi kami - **compiler engine** menggunakan WebSocket."

📍 **`backend/src/main.rs` - Line 63**
```rust
async fn handle_socket(mut socket: WebSocket) {
    // 1. Terima kode dari client
    let code = if let Some(Ok(msg)) = socket.recv().await {
        if let Message::Text(text) = msg {
            if let Ok(req) = serde_json::from_str::<CodeRequest>(&text) {
                req.code
            }
        }
    };
```

> "Pertama, kita menerima kode Rust dari client melalui WebSocket message."

📍 **`backend/src/main.rs` - Line 82**
```rust
let filename = format!("temp/temp_{}.rs", id);
fs::write(&filename, &code).await;
```

> "Kode ditulis ke file temporary dengan UUID unik untuk menghindari konflik."

📍 **`backend/src/main.rs` - Line 99**
```rust
let compile_output = Command::new("rustc")
    .arg(&filename)
    .arg("-o")
    .arg(&exe_name)
    .output()
    .await;
```

> "Kita memanggil compiler `rustc` secara langsung menggunakan Tokio Command untuk async execution."

📍 **`backend/src/main.rs` - Line 144**
```rust
tokio::select! {
    result = stdout_reader.read(&mut stdout_buf) => {
        let text = String::from_utf8_lossy(&stdout_buf[..n]).to_string();
        sender.send(Message::Text(text)).await;
    }
    result = stderr_reader.read(&mut stderr_buf) => {
        // ... kirim error ke client
    }
}
```

> "Output dari program di-stream secara real-time ke client. Baik stdout maupun stderr dikirim melalui WebSocket."

📍 **`backend/src/main.rs` - Line 172**
```rust
while let Some(Ok(msg)) = receiver.next().await {
    if let Message::Text(text) = msg {
        stdin.write_all(input.as_bytes()).await;
    }
}
```

> "Yang unik adalah kita juga menangani **stdin**. Jika program meminta input, user bisa mengetik di browser dan input tersebut dikirim ke program yang sedang berjalan."

📍 **`backend/src/main.rs` - Line 192**
```rust
let _ = fs::remove_file(&filename).await;
let _ = fs::remove_file(&exe_name).await;
```

> "Terakhir, file temporary dibersihkan setelah eksekusi selesai."

---

## BAGIAN 6: FRONTEND - STRUKTUR & ROUTING (Presenter 4)

**[SCENE: Buka file `frontend/src/App.tsx`]**

**Presenter 4:**
> "Sekarang kita beralih ke frontend. Saya akan menjelaskan struktur aplikasi React kami."

📍 **`frontend/src/App.tsx` - Line 22**
```tsx
<Routes>
    <Route path="/login" element={<Login />} />
    <Route path="/register" element={<Register />} />
    <Route path="/" element={<Navigate to="/editor" />} />
    <Route path="/editor" element={<EditorPage />} />
    <Route path="/editor/:id" element={<EditorPage />} />
    <Route 
        path="/dashboard" 
        element={
            <ProtectedRoute>
                <Dashboard />
            </ProtectedRoute>
        } 
    />
</Routes>
```

> "Kami memiliki beberapa route:
> - `/login` dan `/register` untuk autentikasi
> - `/editor` untuk menulis kode baru
> - `/editor/:id` untuk mengedit snippet yang sudah ada
> - `/dashboard` yang **dilindungi** - hanya bisa diakses jika sudah login"

📍 **`frontend/src/App.tsx` - Line 10**
```tsx
const ProtectedRoute: React.FC<{ children: React.ReactNode }> = ({ children }) => {
    const { isAuthenticated } = useAuth();
    if (!isAuthenticated) {
        return <Navigate to="/login" />;
    }
    return <>{children}</>;
};
```

> "ProtectedRoute mengecek status autentikasi. Jika belum login, user akan di-redirect ke halaman login."

---

## BAGIAN 7: FRONTEND - AUTH CONTEXT (Presenter 4)

**[SCENE: Buka file `frontend/src/context/AuthContext.tsx`]**

**Presenter 4:**
> "Untuk state management autentikasi, kami menggunakan React Context."

📍 **`frontend/src/context/AuthContext.tsx` - Line 20**
```tsx
const [token, setToken] = useState<string | null>(localStorage.getItem('token'));

useEffect(() => {
    if (token) {
        axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
    } else {
        delete axios.defaults.headers.common['Authorization'];
    }
}, [token]);
```

> "Token disimpan di localStorage agar user tetap login setelah refresh. Setiap kali token berubah, kita set header Authorization untuk semua request Axios."

📍 **`frontend/src/context/AuthContext.tsx` - Line 37**
```tsx
const login = (newToken: string, username: string) => {
    localStorage.setItem('token', newToken);
    localStorage.setItem('username', username);
    setToken(newToken);
    setUser({ username });
};
```

> "Fungsi login menyimpan token dan username, lalu update state."

---

## BAGIAN 8: FRONTEND - CODE EDITOR (Presenter 1)

**[SCENE: Buka file `frontend/src/pages/Editor.tsx`]**

**Presenter 1:**
> "Halaman editor adalah fitur utama aplikasi kami."

📍 **`frontend/src/pages/Editor.tsx` - Line 213**
```tsx
<Editor
    height="100%"
    defaultLanguage="rust"
    theme="vs-dark"
    value={code}
    onChange={(value) => setCode(value || "")}
    options={{
        minimap: { enabled: false },
        fontSize: 14,
        fontFamily: "'JetBrains Mono', 'Fira Code', monospace",
    }}
/>
```

> "Kami menggunakan **Monaco Editor** - editor yang sama dengan VS Code. Ini memberikan syntax highlighting, auto-completion, dan pengalaman coding yang familiar."

📍 **`frontend/src/pages/Editor.tsx` - Line 72**
```tsx
const handleRun = () => {
    const ws = new WebSocket("ws://localhost:3001/ws");
    socketRef.current = ws;

    ws.onopen = () => {
        ws.send(JSON.stringify({ code }));
    };

    ws.onmessage = (event) => {
        setOutput((prev) => prev + event.data);
    };
};
```

> "Saat user klik Run, kita membuat koneksi WebSocket, mengirim kode, dan menampilkan output secara real-time."

📍 **`frontend/src/pages/Editor.tsx` - Line 97**
```tsx
const handleInput = (e: React.KeyboardEvent<HTMLInputElement>) => {
    if (e.key === "Enter") {
        socketRef.current.send(value + "\n");
        setOutput((prev) => prev + value + "\n");
    }
};
```

> "Jika program meminta input, user bisa mengetik dan menekan Enter. Input dikirim melalui WebSocket ke backend."

📍 **`frontend/src/pages/Editor.tsx` - Line 109**
```tsx
const handleSave = async () => {
    if (id) {
        await axios.put(`http://localhost:3001/snippets/${id}`, { title, code });
    } else {
        const response = await axios.post("http://localhost:3001/snippets", { title, code });
        navigate(`/editor/${response.data.id}`);
    }
};
```

> "User yang sudah login bisa menyimpan kode mereka. Jika ini snippet baru, kita POST. Jika sudah ada, kita PUT untuk update."

---

## BAGIAN 9: FRONTEND - DASHBOARD (Presenter 2)

**[SCENE: Buka file `frontend/src/pages/Dashboard.tsx`]**

**Presenter 2:**
> "Dashboard menampilkan semua snippet yang dimiliki user."

📍 **`frontend/src/pages/Dashboard.tsx` - Line 23**
```tsx
const fetchSnippets = async () => {
    const response = await axios.get('http://localhost:3001/snippets');
    setSnippets(response.data);
};
```

> "Karena token sudah di-set di axios header, request ini otomatis terautentikasi."

📍 **`frontend/src/pages/Dashboard.tsx` - Line 75**
```tsx
{snippets.map((snippet) => (
    <div key={snippet.id} className="bg-gray-900 border border-gray-800 rounded-lg p-6">
        <h3>{snippet.title}</h3>
        <pre>{snippet.code}</pre>
        <Link to={`/editor/${snippet.id}`}>Edit</Link>
        <button onClick={() => handleDelete(snippet.id)}>Delete</button>
    </div>
))}
```

> "Setiap snippet ditampilkan dalam card dengan preview kode, tanggal update, dan tombol untuk edit atau delete."

---

## BAGIAN 10: DEMO LANGSUNG (Semua Presenter)

**[SCENE: Tampilkan aplikasi running di browser]**

**Presenter 3:**
> "Sekarang mari kita lihat demo langsung aplikasi kami."

**[Demo flow:]**
1. **Register** - Buat akun baru
2. **Login** - Masuk dengan akun tersebut
3. **Editor** - Tulis kode Rust sederhana (Hello World)
4. **Run** - Jalankan kode, lihat output real-time
5. **Interactive Input** - Demo program dengan stdin
6. **Save** - Simpan snippet
7. **Dashboard** - Lihat daftar snippet
8. **Edit** - Buka dan edit snippet yang sudah ada

---

## PENUTUP (Presenter 1)

**[SCENE: Semua presenter di frame]**

**Presenter 1:**
> "Itulah presentasi project Compily - Online Rust Compiler dari kami."
>
> "Untuk merangkum, project ini mendemonstrasikan:"
> - Full-stack development dengan Rust dan React
> - Real-time communication dengan WebSocket
> - Secure authentication dengan JWT dan Argon2
> - Modern UI dengan Monaco Editor dan HeroUI
>
> "Terima kasih atas perhatiannya. Apakah ada pertanyaan?"

---

## PEMBAGIAN TUGAS

| Presenter | Bagian |
|-----------|--------|
| **Presenter 1** | Pembukaan, Overview Arsitektur, Code Editor, Penutup |
| **Presenter 2** | Database & State, Autentikasi, Dashboard |
| **Presenter 3** | Snippet Management, Compiler Engine, Demo |
| **Presenter 4** | Frontend Struktur & Routing, Auth Context |

---

## TIPS PRESENTASI

1. **Persiapkan environment** - Pastikan backend dan frontend sudah running sebelum demo
2. **Buat akun test** - Siapkan akun untuk demo agar tidak perlu register on the spot
3. **Siapkan kode contoh** - Beberapa snippet Rust yang menarik untuk demo
4. **Backup plan** - Jika demo gagal, siapkan video recording sebagai backup
5. **Practice** - Latihan beberapa kali agar transisi antar presenter smooth
