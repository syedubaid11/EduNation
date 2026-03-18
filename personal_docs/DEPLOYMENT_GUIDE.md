# 🚀 EduNation Deployment Guide

This guide explains step-by-step how to take EduNation from your local machine to production. We will deploy the **Backend directly to Render** and the **Frontend to Vercel**.

---

## 🛑 Step 1: Prepare the Backend for Production

Before deploying the backend, we must secure the **CORS** (Cross-Origin Resource Sharing) policy. Right now, the backend allows requests from *any* website.

1. **Wait until Step 3** is done to get your actual Vercel Frontend URL (e.g., `https://edunation.vercel.app`).
2. Once you have it, open `backend/src/index.ts` and change this line:
   ```typescript
   // FROM:
   app.use(cors());

   // TO:
   app.use(cors({
     origin: 'https://edunation.vercel.app', // Replace with your ACTUAL Vercel URL
     credentials: true
   }));
   ```
3. Commit and push this change to GitHub.

---

## ☁️ Step 2: Deploy Backend to Render

[Render](https://render.com) is perfect for our Express + TypeScript backend.

1. Go to [Render Dashboard](https://dashboard.render.com).
2. Click **New +** and select **Web Service**.
3. Connect your GitHub account and select the `EduNation` repository.
4. **Configuration Settings:**
   - **Name:** `edunation-api` (or whatever you prefer)
   - **Language:** `Node`
   - **Branch:** `main`
   - **Root Directory:** `backend` 
   - **Build Command:** `npm install`
   - **Start Command:** `npm start` *(This runs `tsx src/index.ts` as defined in package.json)*
5. **Environment Variables:**
   Scroll down and add the EXACT same variables from your `.env` file:
   - `SUPABASE_URL` = `https://your-project.supabase.co`
   - `SUPABASE_ANON_KEY` = `your_super_long_anon_key`
   - `REDIS_URL` = `https://your-database.upstash.io`
   - `REDIS_TOKEN` = `your_redis_token`
   - *(You do NOT need to set PORT, Render handles it automatically)*
6. Click **Create Web Service**. 
7. **Important:** Wait for it to finish building. Once it says "Live", copy the Render URL at the top left (e.g., `https://edunation-api.onrender.com`).

---

## ⚡ Step 3: Deploy Frontend to Vercel

[Vercel](https://vercel.com) provides blazing-fast global deployments for React/Vite apps.

1. Go to [Vercel Dashboard](https://vercel.com/dashboard).
2. Click **Add New** → **Project**.
3. Connect GitHub and import the `EduNation` repository.
4. **Configuration Settings:**
   - **Framework Preset:** `Vite` *(Vercel should auto-detect this)*
   - **Root Directory:** Edit this and select `frontend` instead of the root directory.
   - **Build Command:** `npm run build` *(Auto-detected)*
   - **Output Directory:** `dist` *(Auto-detected)*
5. **Environment Variables:**
   - **Name:** `VITE_API_URL`
   - **Value:** `https://edunation-api.onrender.com/api` *(Paste the Render URL from Step 2, and MAKE SURE you add `/api` at the very end)*
6. Click **Deploy**.
7. Once finished, Vercel will give you a public URL (e.g., `https://edunation.vercel.app`). **Copy this URL.**

---

## 🔗 Step 4: Hook Them Together (The Final Polish)

1. Go back to your code on your local machine.
2. Open `backend/src/index.ts` and do the **Step 1** CORS configuration using the Vercel URL you just copied.
3. Commit the change (`git add . ; git commit -m "chore: secure cors for production" ; git push`).
4. **Render will immediately intercept your push** and auto-deploy the updated backend.

---

## 🎯 Verification Checklist

- [ ] Does visiting the Vercel URL show the 3D globe?
- [ ] If you open your browser Developer Tools (F12) → Network tab, does it successfully fetch data from the Render API without any red CORS errors?
- [ ] Are the metrics loading progressively inside the dashboards?

If yes to all three — **Congratulations, EduNation is live on the internet!** 🎉
