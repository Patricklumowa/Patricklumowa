# Script Video Presentasi: Compily - Online Rust Compiler

## Durasi Estimasi: 10-15 menit

---

## PEMBUKAAN (Jen)

**[SCENE: Semua presenter di frame]**

**Jen:**
> "Halo semuanya! Selamat datang di presentasi project kami yang berjudul **Compily - Online Rust Compiler**."
>
> "Perkenalkan, saya [Jen] dan bersama saya ada:"
> - "[Pat]"
> - "[Kezia]"  
> - "[Eko]"
>
> "Hari ini kami akan mempresentasikan aplikasi web yang memungkinkan pengguna untuk menulis, mengkompilasi, dan menjalankan kode Rust langsung di browser."
>
> "Project ini menggunakan arsitektur full-stack dengan **Backend Rust** menggunakan framework Axum, dan **Frontend React** dengan Vite."

---

## BAGIAN 1: OVERVIEW ARSITEKTUR (Pat)

**[SCENE: Tampilkan diagram arsitektur atau screen share struktur folder]**

**Pat:**
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

## BAGIAN 2: BACKEND - DATABASE & STATE (Pat)

**[SCENE: Buka file `backend/src/db.rs`]**

**Pat:**
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

📍 **`backend/src/db.rs` - Line 26**
```rust
CREATE TABLE IF NOT EXISTS users (
    id TEXT PRIMARY KEY,
    username TEXT UNIQUE NOT NULL,
    password_hash TEXT NOT NULL
);
```

> "Tabel users menyimpan ID, username yang unik, dan password yang sudah di-hash."

📍 **`backend/src/db.rs` - Line 40**
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

## BAGIAN 3: BACKEND - AUTENTIKASI (Pat)

**[SCENE: Buka file `backend/src/auth.rs`]**

**Pat:**
> "Selanjutnya, file `auth.rs` menangani register dan login."

📍 **`backend/src/auth.rs` - Line 20**
```rust
#[derive(Deserialize, ToSchema)]
pub struct RegisterRequest {
    pub username: String,
    pub password: String,
}
```

> "Kita menerima username dan password dari user."

📍 **`backend/src/auth.rs` - Line 101**
```rust
// 2. Hash password
let salt = SaltString::generate(&mut OsRng);
let argon2 = Argon2::default();
let password_hash = argon2
    .hash_password(payload.password.as_bytes(), &salt)
    .map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?
    .to_string();
```

> "Password **tidak pernah disimpan dalam bentuk plain text**. Kami menggunakan algoritma Argon2 untuk hashing, yang merupakan standar industri untuk keamanan password."

📍 **`backend/src/auth.rs` - Line 158**
```rust
let claims = Claims {
    sub: user_id,
    exp: expiration as usize,
};

let secret = std::env::var("JWT_SECRET").unwrap_or_else(|_| "secret".to_string());
let token = encode(&Header::default(), &claims, &EncodingKey::from_secret(secret.as_bytes()))
    .map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?;
```

> "Setelah login berhasil, kita generate JWT token yang berlaku 24 jam. Token ini berisi user ID dan waktu expired."

📍 **`backend/src/auth.rs` - Line 44**
```rust
#[axum::async_trait]
impl<S> FromRequestParts<S> for Claims
where
    S: Send + Sync,
{
    type Rejection = (StatusCode, String);

    async fn from_request_parts(parts: &mut Parts, _state: &S) -> Result<Self, Self::Rejection> {
        // ... extract token from Authorization header
        let token = &auth_header[7..]; // Remove "Bearer "
        let token_data = decode::<Claims>(token, &DecodingKey::from_secret(...));
    }
}
```

> "Yang menarik, Axum memungkinkan kita membuat custom extractor. `Claims` bisa langsung digunakan sebagai parameter fungsi, dan Axum otomatis memvalidasi JWT dari header Authorization."

---

## BAGIAN 4: BACKEND - SNIPPET MANAGEMENT (Eko)

**[SCENE: Buka file `backend/src/snippets.rs`]**

**Eko:**
> "Saya akan menjelaskan manajemen snippet. File ini mengimplementasikan full CRUD operations."

📍 **`backend/src/snippets.rs` - Line 57**
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
        .map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?;

    Ok((StatusCode::CREATED, Json(CreateSnippetResponse { id })))
}
```

> "Perhatikan parameter `claims: Claims`. Ini berarti endpoint ini **membutuhkan autentikasi**. User ID diambil dari JWT, bukan dari request body, sehingga user tidak bisa menyimpan snippet atas nama orang lain."

📍 **`backend/src/snippets.rs` - Line 120**
```rust
let snippet = sqlx::query_as::<_, Snippet>("SELECT * FROM snippets WHERE id = ? AND user_id = ?")
    .bind(&id)
    .bind(&claims.sub)  // memastikan hanya pemilik yang bisa akses
    .fetch_optional(&state.db)
    .await
    .map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?;
```

> "Setiap query snippet selalu menyertakan `user_id` untuk memastikan user hanya bisa mengakses snippet miliknya sendiri."

> "Kami juga mengimplementasikan PATCH untuk partial update - user bisa update title saja atau code saja, tidak harus keduanya."

---

## BAGIAN 5: BACKEND - COMPILER ENGINE (Eko)

**[SCENE: Buka file `backend/src/main.rs`]**

**Eko:**
> "Ini adalah inti dari aplikasi kami - **compiler engine** menggunakan WebSocket."

📍 **`backend/src/main.rs` - Line 64**
```rust
async fn handle_socket(mut socket: WebSocket) {
    // 1. Wait for the first message which should be the code
    let code = if let Some(Ok(msg)) = socket.recv().await {
        if let Message::Text(text) = msg {
            // Try to parse as JSON first, or just take raw text if simple
            if let Ok(req) = serde_json::from_str::<CodeRequest>(&text) {
                req.code
            } else {
                // Fallback if client sends just the code string
                text
            }
        } else {
            return;
        }
    } else {
        return;
    };
```

> "Pertama, kita menerima kode Rust dari client melalui WebSocket message."

📍 **`backend/src/main.rs` - Line 84**
```rust
let filename = format!("temp/temp_{}.rs", id);
let exe_name = if cfg!(target_os = "windows") {
    format!("temp/temp_{}.exe", id)
} else {
    format!("temp/temp_{}", id)
};
```

> "Kode ditulis ke file temporary dengan UUID unik untuk menghindari konflik."

📍 **`backend/src/main.rs` - Line 99**
```rust
// Compile
let compile_output = Command::new("rustc")
    .arg(&filename)
    .arg("-o")
    .arg(&exe_name)
    .output()
    .await;
```

> "Kita memanggil compiler `rustc` secara langsung menggunakan Tokio Command untuk async execution."

📍 **`backend/src/main.rs` - Line 145**
```rust
loop {
    tokio::select! {
        result = stdout_reader.read(&mut stdout_buf) => {
            match result {
                Ok(0) => break, // EOF
                Ok(n) => {
                    let text = String::from_utf8_lossy(&stdout_buf[..n]).to_string();
                    if sender.send(Message::Text(text)).await.is_err() {
                        break;
                    }
                }
                Err(_) => break,
            }
        }
        result = stderr_reader.read(&mut stderr_buf) => {
            // ... kirim error ke client
        }
    }
}
```

> "Output dari program di-stream secara real-time ke client. Baik stdout maupun stderr dikirim melalui WebSocket."

📍 **`backend/src/main.rs` - Line 175**
```rust
// Task to handle WebSocket -> stdin
let mut input_task = tokio::spawn(async move {
    while let Some(Ok(msg)) = receiver.next().await {
        if let Message::Text(text) = msg {
            // Append newline if missing, as read_line usually expects it
            let input = if text.ends_with('\n') { text } else { text + "\n" };
            if stdin.write_all(input.as_bytes()).await.is_err() {
                break;
            }
            if stdin.flush().await.is_err() {
                break;
            }
        } else if let Message::Close(_) = msg {
            break;
        }
    }
});
```

> "Yang unik adalah kita juga menangani **stdin**. Jika program meminta input, user bisa mengetik di browser dan input tersebut dikirim ke program yang sedang berjalan."

📍 **`backend/src/main.rs` - Line 207**
```rust
// Cleanup
let _ = fs::remove_file(&filename).await;
let _ = fs::remove_file(&exe_name).await;
if cfg!(target_os = "windows") {
    let pdb_name = format!("temp/temp_{}.pdb", id);
    let _ = fs::remove_file(&pdb_name).await;
}
```

> "Terakhir, file temporary dibersihkan setelah eksekusi selesai."

---

## BAGIAN 6: FRONTEND - STRUKTUR & ROUTING (Jen)

**[SCENE: Buka file `frontend/src/App.tsx`]**

**Jen:**
> "Sekarang kita beralih ke frontend. Saya akan menjelaskan struktur aplikasi React kami."

📍 **`frontend/src/App.tsx` - Line 23**
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

📍 **`frontend/src/App.tsx` - Line 9**
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

## BAGIAN 7: FRONTEND - AUTH CONTEXT (Jen)

**[SCENE: Buka file `frontend/src/context/AuthContext.tsx`]**

**Jen:**
> "Untuk state management autentikasi, kami menggunakan React Context."

📍 **`frontend/src/context/AuthContext.tsx` - Line 19**
```tsx
const [user, setUser] = useState<User | null>(null);
const [token, setToken] = useState<string | null>(localStorage.getItem('token'));

useEffect(() => {
  if (token) {
    // Set default axios header
    axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
  } else {
    delete axios.defaults.headers.common['Authorization'];
  }
}, [token]);
```

> "Token disimpan di localStorage agar user tetap login setelah refresh. Setiap kali token berubah, kita set header Authorization untuk semua request Axios."

📍 **`frontend/src/context/AuthContext.tsx` - Line 39**
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

## BAGIAN 8: FRONTEND - CODE EDITOR (Kezia)

**[SCENE: Buka file `frontend/src/pages/Editor.tsx`]**

**Kezia:**
> "Halaman editor adalah fitur utama aplikasi kami."

📍 **`frontend/src/pages/Editor.tsx` - Line 231**
```tsx
<Editor
    height="100%"
    width="100%"
    defaultLanguage="rust"
    theme="vs-dark"
    value={code}
    onChange={(value) => setCode(value || "")}
    options={{
        minimap: { enabled: false },
        fontSize: 14,
        fontFamily: "'JetBrains Mono', 'Fira Code', monospace",
        scrollBeyondLastLine: false,
        automaticLayout: true,
    }}
/>
```

> "Kami menggunakan **Monaco Editor** - editor yang sama dengan VS Code. Ini memberikan syntax highlighting, auto-completion, dan pengalaman coding yang familiar."

📍 **`frontend/src/pages/Editor.tsx` - Line 74**
```tsx
const handleRun = () => {
  if (socketRef.current) {
    socketRef.current.close();
  }

  setIsLoading(true);
  setOutput("");

  const ws = new WebSocket("ws://localhost:3001/ws");
  socketRef.current = ws;

  ws.onopen = () => {
    // Send the code as the first message
    ws.send(JSON.stringify({ code }));
  };

  ws.onmessage = (event) => {
    const data = event.data;
    setOutput((prev) => prev + data);
  };

  ws.onclose = () => {
    setIsLoading(false);
    socketRef.current = null;
  };
};
```

> "Saat user klik Run, kita membuat koneksi WebSocket, mengirim kode, dan menampilkan output secara real-time."

📍 **`frontend/src/pages/Editor.tsx` - Line 104**
```tsx
const handleInput = (e: React.KeyboardEvent<HTMLInputElement>) => {
  if (e.key === "Enter") {
    const value = e.currentTarget.value;
    if (socketRef.current && socketRef.current.readyState === WebSocket.OPEN) {
      socketRef.current.send(value + "\n");
      // Local echo
      setOutput((prev) => prev + value + "\n");
      e.currentTarget.value = "";
    }
  }
};
```

> "Jika program meminta input, user bisa mengetik dan menekan Enter. Input dikirim melalui WebSocket ke backend."

📍 **`frontend/src/pages/Editor.tsx` - Line 118**
```tsx
const handleSave = async () => {
  if (!isAuthenticated) {
    alert("Please login to save snippets");
    navigate("/login");
    return;
  }

  setIsSaving(true);
  try {
    if (id) {
      await axios.put(`http://localhost:3001/snippets/${id}`, {
        title,
        code,
        language: "rust"
      });
    } else {
      const response = await axios.post("http://localhost:3001/snippets", {
        title,
        code,
        language: "rust"
      });
      navigate(`/editor/${response.data.id}`);
    }
    alert("Snippet saved successfully!");
  } catch (err) {
    console.error("Failed to save snippet", err);
    alert("Failed to save snippet");
  } finally {
    setIsSaving(false);
  }
};
```

> "User yang sudah login bisa menyimpan kode mereka. Jika ini snippet baru, kita POST. Jika sudah ada, kita PUT untuk update."

---

## BAGIAN 9: FRONTEND - DASHBOARD (Kezia)

**[SCENE: Buka file `frontend/src/pages/Dashboard.tsx`]**

**Kezia:**
> "Dashboard menampilkan semua snippet yang dimiliki user."

📍 **`frontend/src/pages/Dashboard.tsx` - Line 24**
```tsx
const fetchSnippets = async () => {
  try {
    const response = await axios.get('http://localhost:3001/snippets');
    setSnippets(response.data);
    setLoading(false);
  } catch (err) {
    setError('Failed to fetch snippets');
    setLoading(false);
  }
};
```

> "Karena token sudah di-set di axios header, request ini otomatis terautentikasi."

📍 **`frontend/src/pages/Dashboard.tsx` - Line 77**
```tsx
{snippets.map((snippet) => (
  <div key={snippet.id} className="bg-gray-900 border border-gray-800 rounded-lg p-6 hover:border-gray-700 transition-colors">
    <div className="flex justify-between items-start mb-4">
      <h3 className="text-xl font-semibold text-white truncate pr-4">{snippet.title}</h3>
    </div>
    
    <div className="bg-gray-950 rounded p-3 mb-4 h-32 overflow-hidden relative">
      <pre className="text-gray-400 text-xs font-mono">
        {snippet.code}
      </pre>
    </div>
    
    <div className="flex justify-between items-center text-sm text-gray-500">
      <span>{new Date(snippet.updated_at).toLocaleDateString()}</span>
      <div className="flex gap-3">
        <Link to={`/editor/${snippet.id}`}>Edit</Link>
        <button onClick={() => handleDelete(snippet.id)}>Delete</button>
      </div>
    </div>
  </div>
))}
```

> "Setiap snippet ditampilkan dalam card dengan preview kode, tanggal update, dan tombol untuk edit atau delete."

---

## BAGIAN 10: DEMO LANGSUNG (Semua Presenter)

**[SCENE: Tampilkan aplikasi running di browser]**

**Jen:**
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

## PENUTUP (Jen)

**[SCENE: Semua presenter di frame]**

**Jen:**
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



---


