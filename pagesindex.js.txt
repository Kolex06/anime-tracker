// Full-featured Anime Tracking Site using Next.js, Tailwind, Firebase, and Jikan API
// Install dependencies: npm install next react react-dom tailwindcss @tailwindcss/forms axios swr firebase

import { useState, useEffect } from 'react';
import useSWR from 'swr';
import axios from 'axios';
import { getAuth, signInWithPopup, GoogleAuthProvider, signOut } from "firebase/auth";
import { initializeApp } from "firebase/app";
import { getFirestore, collection, addDoc, getDocs, deleteDoc, doc } from "firebase/firestore";

const firebaseConfig = {
  apiKey: "YOUR_API_KEY",
  authDomain: "YOUR_AUTH_DOMAIN",
  projectId: "YOUR_PROJECT_ID",
  storageBucket: "YOUR_STORAGE_BUCKET",
  messagingSenderId: "YOUR_MESSAGING_SENDER_ID",
  appId: "YOUR_APP_ID"
};

const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const provider = new GoogleAuthProvider();

const fetcher = (url) => axios.get(url).then((res) => res.data);

export default function Home() {
  const { data, error } = useSWR('https://api.jikan.moe/v4/top/anime', fetcher);
  const [watchlist, setWatchlist] = useState([]);
  const [user, setUser] = useState(null);

  useEffect(() => {
    auth.onAuthStateChanged((currentUser) => {
      setUser(currentUser);
      if (currentUser) loadWatchlist(currentUser.uid);
    });
  }, []);

  const login = async () => {
    const result = await signInWithPopup(auth, provider);
    setUser(result.user);
    loadWatchlist(result.user.uid);
  };

  const logout = async () => {
    await signOut(auth);
    setUser(null);
    setWatchlist([]);
  };

  const loadWatchlist = async (userId) => {
    const querySnapshot = await getDocs(collection(db, `watchlists/${userId}/anime`));
    const animeList = querySnapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
    setWatchlist(animeList);
  };

  const addToWatchlist = async (anime) => {
    if (!user) return alert("Please log in to track anime");
    const docRef = await addDoc(collection(db, `watchlists/${user.uid}/anime`), anime);
    setWatchlist([...watchlist, { id: docRef.id, ...anime }]);
  };

  const removeFromWatchlist = async (id) => {
    if (!user) return;
    await deleteDoc(doc(db, `watchlists/${user.uid}/anime`, id));
    setWatchlist(watchlist.filter(a => a.id !== id));
  };

  if (error) return <div>Failed to load anime list</div>;
  if (!data) return <div>Loading...</div>;

  return (
    <div className="container mx-auto p-4">
      <h1 className="text-3xl font-bold mb-4">Anime Tracking Site</h1>
      {user ? (
        <button onClick={logout} className="bg-red-500 text-white px-4 py-2 rounded">Logout</button>
      ) : (
        <button onClick={login} className="bg-green-500 text-white px-4 py-2 rounded">Login with Google</button>
      )}
      <h2 className="text-2xl mt-4">Top Anime</h2>
      <div className="grid grid-cols-2 md:grid-cols-4 gap-4">
        {data.data.map((anime) => (
          <div key={anime.mal_id} className="border p-2 rounded shadow">
            <img src={anime.images.jpg.image_url} alt={anime.title} className="w-full h-40 object-cover" />
            <h3 className="text-lg font-semibold mt-2">{anime.title}</h3>
            <button
              onClick={() => addToWatchlist(anime)}
              className="mt-2 bg-blue-500 text-white px-4 py-2 rounded"
            >
              Add to Watchlist
            </button>
          </div>
        ))}
      </div>
      <h2 className="text-2xl mt-6">My Watchlist</h2>
      <ul>
        {watchlist.map((anime) => (
          <li key={anime.id} className="flex justify-between border p-2 rounded mt-2">
            {anime.title}
            <button onClick={() => removeFromWatchlist(anime.id)} className="bg-red-500 text-white px-2 py-1 rounded">Remove</button>
          </li>
        ))}
      </ul>
    </div>
  );
}
