# Deployment Guide: Mini Social Post App

Deploy frontend to Vercel and backend to Render with MongoDB Atlas.

---
## 1. Prerequisites
- GitHub repository containing this project root (`MiniSocialApp/` with `backend/` and `frontend/` folders).
- MongoDB Atlas cluster and database (you already have a URI in `.env.example`).
- Accounts: Vercel + Render + GitHub.

---
## 2. Environment Variables
Backend (Render):
- `MONGODB_URI` = your Atlas SRV URI
- `JWT_SECRET` = a strong random string (Render can auto-generate; replace if desired)
- `PORT` (Render will inject `PORT`, Express should use `process.env.PORT || 4000`).

Frontend (Vercel):
- `REACT_APP_API_URL` = Render backend public URL (e.g. `https://mini-social-backend.onrender.com`)

After first backend deploy, copy its URL and set in Vercel before building frontend.

---
## 3. Backend Deployment (Render)
1. Commit `render.yaml` (already added). This allows Render "Blueprint" deployment OR you can deploy service manually.
2. In Render dashboard:
   - New + "Blueprint" → connect repo → pick `render.yaml`.
   - OR New Web Service → Root directory: `backend`.
3. Runtime: Node 18 or 20.
4. Build command: `npm install`.
5. Start command: `npm start`.
6. Add Disk:
   - Name: uploads
   - Mount Path: `/opt/render/project/src/backend/uploads` (must match static path) – already in `render.yaml`.
7. Add env vars:
   - `MONGODB_URI`
   - `JWT_SECRET`
8. Deploy. Watch logs for `Server listening` and `MongoDB connected`.
9. Test health endpoint: `curl https://<service-url>/api/health` should return JSON `{ status: 'ok', ... }`.

### CORS
If frontend hosted on Vercel, optionally restrict CORS in `backend/server.js`:
```js
app.use(cors({ origin: [process.env.FRONTEND_ORIGIN], credentials: false }));
```
Add `FRONTEND_ORIGIN` to env after you know the Vercel domain.

---
## 4. Frontend Deployment (Vercel)
1. Commit `frontend/vercel.json` (added).
2. Push repository to GitHub.
3. In Vercel dashboard → New Project → Import GitHub → Select repo.
4. Root directory: choose `frontend`.
5. Framework preset: Create React App (auto-detect via `react-scripts`).
6. Set Environment Variable:
   - `REACT_APP_API_URL` = backend Render URL.
7. Build Output Directory: `build` (in `vercel.json` config). Build command will be auto: `npm run build`.
8. Deploy → Wait for success.
9. Visit domain → ensure requests hit backend (check Network tab for 200 on `/api/auth/login`).

---
## 5. Uploads Path & Images
Express serves: `app.use('/uploads', express.static(path.join(__dirname, 'uploads')));`
- With Render disk mount, images persist across deploys.
- Frontend image URL logic prefixes relative paths with `REACT_APP_API_URL`.

---
## 6. Post-Deploy Checks
Backend:
- `/api/health` 200
- Signup/Login returns token
- Create Post stores image (verify file saved on disk by listing directory via Render Shell).

Frontend:
- Environment variable correct (check caption in Signup page if still present).
- Like/comment operations succeed (401 only when token missing).
- Dark/light toggle works server-side static build.

---
## 7. Common Issues
| Issue | Cause | Fix |
|-------|-------|-----|
| 404 on images | Wrong base URL | Ensure `REACT_APP_API_URL=https://<render-domain>` |
| CORS error | Missing allowed origin | Add cors origin array in backend | 
| 500 Mongo connect | Bad URI or IP access | Confirm Atlas Network Access allows all (0.0.0.0/0) or Render egress IP |
| JWT invalid after deploy | Old secret mismatch | Keep consistent `JWT_SECRET` value |

---
## 8. Optional Improvements
- Add rate limiting (e.g. `express-rate-limit`).
- Add helmet for security headers.
- Add Vercel Analytics or simple Plausible script.
- Configure Render auto-scaling if upgrading plan.

---
## 9. Manual Test Commands
```powershell
# Health
curl https://mini-social-backend.onrender.com/api/health

# Signup
curl -X POST https://mini-social-backend.onrender.com/api/auth/signup \ 
  -H "Content-Type: application/json" \ 
  -d '{"username":"demo","email":"demo@example.com","password":"StrongPass123"}'

# Login
curl -X POST https://mini-social-backend.onrender.com/api/auth/login \ 
  -H "Content-Type: application/json" \ 
  -d '{"email":"demo@example.com","password":"StrongPass123"}'
```
(Adjust domain & credentials.)

---
## 10. Rollback Strategy
- Use Render deploy list to promote previous successful version.
- For frontend, redeploy older commit from Vercel dashboard.

---
## 11. Environment Variable Summary
| Service | Key | Value |
|---------|-----|-------|
| Render backend | MONGODB_URI | Atlas SRV URI |
| Render backend | JWT_SECRET | Random 64 hex |
| Vercel frontend | REACT_APP_API_URL | Render backend URL |
| (optional) | FRONTEND_ORIGIN | Vercel domain for stricter CORS |

---
## 12. Security Notes
- Never commit real secrets to GitHub (`.env.example` is fine). Remove actual credentials before pushing.
- Rotate JWT secret if leaked.
- Consider enabling MongoDB IP restrictions + Render fixed egress IP if moving to paid plan.

Happy deploying! 🚀
