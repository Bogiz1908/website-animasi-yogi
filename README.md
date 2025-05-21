# website-animasi-yogi<script type="module">
        // Impor fungsi yang diperlukan dari Firebase SDK
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js"; // signInWithCustomToken dihapus jika tidak ada backend
        import { getFirestore, collection, addDoc, serverTimestamp, setLogLevel } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // ----------------------------------------------------------------------------------
        // !!! PENTING: GANTI DENGAN KONFIGURASI FIREBASE PROYEK ANDA SENDIRI DI BAWAH INI !!!
        // ----------------------------------------------------------------------------------
        const firebaseConfig = {
            apiKey: "AIzaSyXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX", // Ganti dengan API Key Anda
            authDomain: "nama-proyek-anda.firebaseapp.com",    // Ganti dengan Auth Domain Anda
            projectId: "nama-proyek-anda",                     // Ganti dengan Project ID Anda
            storageBucket: "nama-proyek-anda.appspot.com",     // Ganti dengan Storage Bucket Anda
            messagingSenderId: "000000000000",                 // Ganti dengan Messaging Sender ID Anda
            appId: "1:000000000000:web:xxxxxxxxxxxxxxxxxxxxxx", // Ganti dengan App ID Anda
            // measurementId: "G-XXXXXXXXXX" // Opsional, jika Anda menggunakan Google Analytics
        };
        // ----------------------------------------------------------------------------------

        const appId = 'website-yogi-pravda'; // Anda bisa gunakan nama repositori atau ID unik lain

        let db, auth;
        let currentUserId = null;

        if (firebaseConfig && firebaseConfig.apiKey !== "AIzaSyXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX") { // Cek apakah config sudah diisi
            try {
                const app = initializeApp(firebaseConfig);
                db = getFirestore(app);
                auth = getAuth(app);
                setLogLevel('debug');

                const authenticateUser = async () => {
                    try {
                        console.log("Attempting to sign in anonymously for GitHub Pages...");
                        await signInAnonymously(auth);
                        console.log("Signed in anonymously on GitHub Pages.");
                    } catch (error) {
                        console.error("Anonymous authentication error on GitHub Pages:", error);
                        const userIdDisplay = document.getElementById('userIdDisplay');
                        if (userIdDisplay) {
                             userIdDisplay.textContent = `Authentication Error: ${error.message}. Firestore features might be limited.`;
                        }
                    }
                };

                onAuthStateChanged(auth, (user) => {
                    const userIdDisplay = document.getElementById('userIdDisplay');
                    if (user) {
                        currentUserId = user.uid;
                        console.log("User authenticated with UID:", currentUserId);
                        if (userIdDisplay) {
                            userIdDisplay.textContent = `User ID: ${currentUserId}`;
                        }
                    } else {
                        currentUserId = crypto.randomUUID();
                        console.log("User not authenticated. Using random ID for this session:", currentUserId);
                         if (userIdDisplay) {
                            userIdDisplay.textContent = `User ID (anonymous session): ${currentUserId}`;
                        }
                    }
                });

                authenticateUser();
            } catch (e) {
                console.error("Error initializing Firebase:", e);
                 const userIdDisplay = document.getElementById('userIdDisplay');
                if (userIdDisplay) {
                    userIdDisplay.textContent = "Error initializing Firebase. Contact form may not work.";
                }
            }

        } else {
            console.error("Firebase config is not properly set up in the script. Firestore functionality will be disabled.");
            const userIdDisplay = document.getElementById('userIdDisplay');
            if (userIdDisplay) {
                userIdDisplay.textContent = "Firebase is not configured. Contact form will not work.";
            }
        }

        // ... (sisa kode JavaScript Anda untuk animasi, scroll, dan form handling tetap sama) ...
        // Pastikan event listener untuk form submit menggunakan variabel db dan auth yang sudah diinisialisasi di atas.

        function isTopInViewport(element) {
            const rect = element.getBoundingClientRect();
            return rect.top <= (window.innerHeight || document.documentElement.clientHeight) && rect.bottom >=0;
        }

        function handleScrollAnimations() {
            const elements = document.querySelectorAll('.fade-in');
            elements.forEach(el => {
                if (isTopInViewport(el)) {
                    el.classList.add('visible');
                }
            });
        }

        window.addEventListener('scroll', handleScrollAnimations);
        document.addEventListener('DOMContentLoaded', () => {
            handleScrollAnimations();
            document.getElementById('currentYear').textContent = new Date().getFullYear();

            document.querySelectorAll('a[href^="#"]').forEach(anchor => {
                anchor.addEventListener('click', function (e) {
                    e.preventDefault();
                    const targetId = this.getAttribute('href');
                    const targetElement = document.querySelector(targetId);
                    if(targetElement) {
                        targetElement.scrollIntoView({
                            behavior: 'smooth'
                        });
                    }
                });
            });

            const contactForm = document.getElementById('contactForm');
            const formStatusDiv = document.getElementById('formStatus');
            const submitButton = document.getElementById('submitContactForm');

            if (contactForm) { // Cek apakah form ada
                contactForm.addEventListener('submit', async (e) => {
                    e.preventDefault();

                    if (!db || !auth || !currentUserId) { // Cek apakah Firebase siap dan User ID ada
                        console.error("Firebase not ready or User ID not available. Cannot submit form.");
                        formStatusDiv.textContent = 'Error: Layanan formulir belum siap. Silakan coba lagi nanti.';
                        formStatusDiv.className = 'form-status-message error';
                        formStatusDiv.style.display = 'block';
                        return;
                    }

                    submitButton.disabled = true;
                    submitButton.textContent = 'Mengirim...';
                    formStatusDiv.style.display = 'none';

                    const name = contactForm.name.value.trim();
                    const email = contactForm.email.value.trim();
                    const message = contactForm.message.value.trim();

                    if (!name || !email || !message) {
                        formStatusDiv.textContent = 'Semua field wajib diisi.';
                        formStatusDiv.className = 'form-status-message error';
                        formStatusDiv.style.display = 'block';
                        submitButton.disabled = false;
                        submitButton.textContent = 'Kirim Pesan';
                        return;
                    }
                    
                    const messagesCollectionPath = `artifacts/${appId}/users/${currentUserId}/contact_messages`;

                    try {
                        await addDoc(collection(db, messagesCollectionPath), {
                            name: name,
                            email: email,
                            message: message,
                            createdAt: serverTimestamp()
                        });
                        formStatusDiv.textContent = 'Pesan berhasil terkirim! Terima kasih.';
                        formStatusDiv.className = 'form-status-message success';
                        contactForm.reset();
                    } catch (error) {
                        console.error("Error adding document: ", error);
                        formStatusDiv.textContent = 'Terjadi kesalahan saat mengirim pesan. Silakan coba lagi.';
                        formStatusDiv.className = 'form-status-message error';
                    } finally {
                        formStatusDiv.style.display = 'block';
                        submitButton.disabled = false;
                        submitButton.textContent = 'Kirim Pesan';
                    }
                });
            }
             // Logika jika Firebase tidak terkonfigurasi dengan benar untuk form
            if (!db || !auth) {
                 if (formStatusDiv) {
                    formStatusDiv.textContent = 'Fungsi kontak dinonaktifkan karena Firebase tidak terkonfigurasi dengan benar.';
                    formStatusDiv.className = 'form-status-message error';
                    formStatusDiv.style.display = 'block';
                 }
                 if (submitButton) submitButton.disabled = true;
            }
        });
    </script>

</body>
</html>
