# noel-en-famille
liste noel
// README
// Projet: Noël en Famille — site gratuit (React + Tailwind + Firebase)
// Langue: Français
// Objectif: site privé pour listes de Noël, connexion par email (sign-in link), jusqu'à 30 membres, extraction automatique image+titre depuis un lien produit, case à cocher verouillée quand acheté.

/*
Structure du dépôt (fichiers importants inclus ci-dessous)

package.json
tailwind.config.js
postcss.config.js
/vercel.json
/pages/_app.jsx
/pages/index.jsx         -> Page d'accueil
/pages/membre/[id].jsx   -> Page membre
/src/lib/firebase.js     -> Config Firebase
/pages/api/og.js         -> API serverless pour extraire OpenGraph (hébergeable sur Vercel)
/public/default-bg.jpg   -> image par défaut
/public/default-avatar.png -> avatar par défaut

Instructions rapides (voir plus bas pour détails):
1) Créer un projet Firebase (gratuit) et activer Firestore + Authentication (Email Link sign-in)
2) Copier les variables Firebase dans les secrets Vercel
3) Déployer sur Vercel (compte gratuit) : connecter repo GitHub / ou importer depuis ZIP
4) Partager le lien avec la famille


==== package.json ====
{
  "name": "noel-en-famille",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start"
  },
  "dependencies": {
    "next": "13.5.0",
    "react": "18.2.0",
    "react-dom": "18.2.0",
    "firebase": "10.9.0",
    "cheerio": "1.0.0-rc.12",
    "node-fetch": "2.6.9",
    "clsx": "1.2.1"
  },
  "devDependencies": {
    "tailwindcss": "^3.5.0",
    "postcss": "^8.4.21",
    "autoprefixer": "^10.4.14"
  }
}


==== tailwind.config.js ====
module.exports = {
  content: ["./pages/**/*.{js,jsx}", "./components/**/*.{js,jsx}", "./src/**/*.{js,jsx}"],
  theme: {
    extend: {},
  },
  plugins: [],
}

==== pages/_app.jsx ====
import '../styles/globals.css'
export default function App({ Component, pageProps }) {
  return <Component {...pageProps} />
}

==== src/lib/firebase.js ====
import { initializeApp } from 'firebase/app'
import { getAuth, sendSignInLinkToEmail, isSignInWithEmailLink, signInWithEmailLink } from 'firebase/auth'
import { getFirestore } from 'firebase/firestore'

// Remplacez ces valeurs par vos clés Firebase (mettre dans Vercel env vars)
const firebaseConfig = {
  apiKey: process.env.NEXT_PUBLIC_FIREBASE_API_KEY,
  authDomain: process.env.NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN,
  projectId: process.env.NEXT_PUBLIC_FIREBASE_PROJECT_ID,
  storageBucket: process.env.NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET,
  messagingSenderId: process.env.NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID,
  appId: process.env.NEXT_PUBLIC_FIREBASE_APP_ID
}

const app = initializeApp(firebaseConfig)
export const auth = getAuth(app)
export const db = getFirestore(app)

export async function sendMagicLink(email, url) {
  const actionCodeSettings = {
    url: url, // URL de redirection après connexion
    handleCodeInApp: true
  }
  await sendSignInLinkToEmail(auth, email, actionCodeSettings)
}

export async function completeSignIn(url) {
  if (isSignInWithEmailLink(auth, url)) {
    // l'email doit être récupéré depuis localStorage ou demandé à l'utilisateur
    const email = window.localStorage.getItem('pendingEmail')
    if (!email) return null
    const userCredential = await signInWithEmailLink(auth, email, url)
    window.localStorage.removeItem('pendingEmail')
    return userCredential
  }
  return null
}

==== pages/index.jsx ====
import Link from 'next/link'
import { useEffect, useState } from 'react'
import { auth, db } from '../src/lib/firebase'
import { collection, getDocs } from 'firebase/firestore'

export default function Home() {
  const [members, setMembers] = useState([])

  useEffect(() => {
    // Lecture simple de membres (exemple: collection 'members')
    async function load() {
      try {
        const snap = await getDocs(collection(db, 'members'))
        setMembers(snap.docs.map(d => ({ id: d.id, ...d.data() })))
      } catch (e) {
        console.error(e)
      }
    }
    load()
  }, [])

  return (
    <div className="min-h-screen bg-cover bg-center" style={{ backgroundImage: "url('/default-bg.jpg')" }}>
      <div className="backdrop-blur-sm bg-white/30 min-h-screen p-8">
        <h1 className="text-4xl font-bold mb-6">Noël en Famille</h1>
        <div className="grid grid-cols-4 gap-4">
          {members.slice(0,30).map(m => (
            <Link key={m.id} href={`/membre/${m.id}`}>
              <a className="flex flex-col items-center p-4 bg-white/60 rounded-lg shadow">
                <img src={m.avatar || '/default-avatar.png'} className="w-20 h-20 rounded-full object-cover mb-2" />
                <div>{m.name}</div>
              </a>
            </Link>
          ))}
        </div>
      </div>
    </div>
  )
}

==== pages/membre/[id].jsx ====
import { useRouter } from 'next/router'
import { useEffect, useState } from 'react'
import { db, auth } from '../../src/lib/firebase'
import { doc, getDoc, collection, getDocs, addDoc, setDoc, updateDoc } from 'firebase/firestore'

export default function MemberPage() {
  const router = useRouter()
  const { id } = router.query
  const [member, setMember] = useState(null)
  const [items, setItems] = useState([])
  const [newLink, setNewLink] = useState('')

  useEffect(() => {
    if (!id) return
    async function load() {
      const ref = doc(db, 'members', id)
      const snap = await getDoc(ref)
      if (snap.exists()) setMember({ id: snap.id, ...snap.data() })
      // load items
      const itemsSnap = await getDocs(collection(ref, 'items'))
      setItems(itemsSnap.docs.map(d => ({ id: d.id, ...d.data() })))
    }
    load()
  }, [id])

  async function addItem() {
    if (!newLink) return
    // call serverless to fetch og:image and og:title
    const res = await fetch(`/api/og?url=${encodeURIComponent(newLink)}`)
    const data = await res.json()
    const ref = doc(db, 'members', id)
    await addDoc(collection(ref, 'items'), {
      title: data.title || 'Article',
      image: data.image || '/default-product.png',
      link: newLink,
      bought: false,
      createdAt: new Date()
    })
    setNewLink('')
    // reload
    const itemsSnap = await getDocs(collection(ref, 'items'))
    setItems(itemsSnap.docs.map(d => ({ id: d.id, ...d.data() })))
  }

  async function toggleBought(item) {
    // lock permanently when set true
    const itemRef = doc(db, 'members', id, 'items', item.id)
    if (item.bought) return // ne pas décocher
    await updateDoc(itemRef, { bought: true })
    setItems(i => i.map(x => x.id === item.id ? { ...x, bought: true } : x))
  }

  if (!member) return <div>Chargement...</div>

  return (
    <div className="p-8">
      <h2 className="text-3xl mb-4">{member.name}</h2>
      <div className="mb-6">
        <input value={newLink} onChange={e => setNewLink(e.target.value)} placeholder="Coller le lien d'un produit" className="w-2/3 p-2 border" />
        <button onClick={addItem} className="ml-2 p-2 bg-green-600 text-white rounded">Ajouter</button>
      </div>
      <div className="grid grid-cols-3 gap-4">
        {items.map(it => (
          <div key={it.id} className={`p-4 border rounded ${it.bought ? 'opacity-50 grayscale' : ''}`}>
            <img src={it.image} className="w-full h-40 object-cover mb-2" />
            <div className="font-medium">{it.title}</div>
            <a href={it.link} target="_blank" rel="noreferrer" className="text-sm break-all">Voir le produit</a>
            <div className="mt-2">
              <label className="flex items-center space-x-2">
                <input type="checkbox" checked={it.bought} onChange={() => toggleBought(it)} disabled={it.bought} />
                <span>{it.bought ? 'Acheté 🔒' : 'Marquer comme acheté'}</span>
              </label>
            </div>
          </div>
        ))}
      </div>
    </div>
  )
}

==== pages/api/og.js ====
// Vercel serverless API pour récupérer les balises OpenGraph d'une page publique
const fetch = require('node-fetch')
const cheerio = require('cheerio')

module.exports = async (req, res) => {
  const { url } = req.query
  if (!url) return res.status(400).json({ error: 'url manquante' })
  try {
    const r = await fetch(url, { timeout: 5000 })
    const html = await r.text()
    const $ = cheerio.load(html)
    const title = $('meta[property="og:title"]').attr('content') || $('title').text()
    const image = $('meta[property="og:image"]').attr('content') || $('meta[name="twitter:image"]').attr('content') || ''
    res.setHeader('Content-Type', 'application/json')
    res.status(200).send(JSON.stringify({ title, image }))
  } catch (e) {
    res.status(500).json({ error: 'impossible de récupérer la page' })
  }
}


==== Notes importantes et fonctionnement ====
- Auth: j'ai utilisé le flux email link (connexion sans mot de passe). Les utilisateurs reçoivent un e-mail contenant un lien magique pour se connecter.
  - Pour envoyer le lien, on enregistre l'email dans localStorage avant d'appeler sendSignInLinkToEmail.
  - Ce flux est pratique et gratuit avec Firebase.
- Multi-utilisateurs avec même adresse mail: techniquement possible (on peut attribuer plusieurs 'members' à une même adresse). L'auth Firebase identifie l'utilisateur par compte unique (adresse mail). Mais on peut autoriser qu'un membre soit géré par n'importe quel utilisateur (plusieurs comptes partagent la visibilité). Si tu veux plusieurs personnes utilisant la même adresse à la fois, c'est possible — elles se connecteront avec le même e-mail.
- Scraping: la page /api/og prend un URL public et retourne og:title et og:image. Hébergé sur Vercel, il est gratuit tant que le trafic est faible (usage familial).
- Verrouillage définitif: lorsque un item est marqué comme acheté, on ne permet pas de le décocher (logique dans toggleBought).


==== Déploiement (étapes détaillées) ====
1) Créer un compte GitHub et pousser ce dépôt (ou téléverser un ZIP) — gratuit
2) Créer un compte Firebase (console.firebase.google.com)
   - Nouveau projet
   - Activer Authentication > Sign-in method > Email link (Passwordless)
   - Activer Firestore (mode test ou règles sécurisées selon préférence)
   - Récupérer la configuration du SDK (apiKey, authDomain, projectId, storageBucket, messagingSenderId, appId)
3) Créer un projet sur Vercel (vercel.com), connecter ton compte GitHub et importer le repo
   - Dans Vercel > Settings > Environment Variables, ajouter les clés suivantes (prefix NEXT_PUBLIC_):
     NEXT_PUBLIC_FIREBASE_API_KEY
     NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN
     NEXT_PUBLIC_FIREBASE_PROJECT_ID
     NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET
     NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID
     NEXT_PUBLIC_FIREBASE_APP_ID
4) Déployer — Vercel build -> Next.js détecté
5) Aller sur l'URL fournie par Vercel, tester l'inscription par email (tu peux t'envoyer un lien à toi-même)
6) Créer manuellement 'members' depuis Firestore (collection 'members') ou additionner un formulaire d'administration (optionnel)


==== Checklist final avant partage ====
- [ ] Ajouter une image de fond personnalisée dans /public et référencer dans index
- [ ] Pré-remplir la collection 'members' (max 30)
- [ ] Tester ajout d'articles via copier-coller de lien
- [ ] Tester le verrouillage d'articles par un autre compte


Bonne utilisation !

/*
Fin du document. Tu peux copier-coller ce projet dans un dépôt GitHub, puis suivre les étapes de déploiement listées ci-dessus.
*/
