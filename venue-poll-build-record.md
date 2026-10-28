# Build Record — SHAPE-SEA 2026 Retreat Venue Poll

Purpose: a record of what the poll web app does, so it can be rebuilt and hosted **outside Claude**.
Current live version (Claude-hosted): https://claude.ai/artifact/R6JSUb2pkkGTaEPV3t5R9E
Source file: `venue-poll-en.html` (single file, images embedded). Images also exported in `venue-images/`.

## 1. Event
SHAPE-SEA 2026 Midterm Review Retreat and MEAL Refresher Workshop — 20–22 Nov 2026.
Question: where should it be held?

## 2. Venues (source: M_E_-_PLACE.pdf)
| id | English name | Thai name | Area | Website | Image |
|---|---|---|---|---|---|
| neera | Neera Retreat Hotel | นีรา รีทรีท โฮเทล | Samphran, Nakhon Pathom | https://www.neeraretreathotel.com | venue-images/1-neera.jpg |
| riverton | Riverton Hotel Amphawa | โรงแรมริเวอร์ตัน อัมพวา | Amphawa, Samut Songkhram | https://rivertonhotel.th-thailand.com/en/ | venue-images/2-riverton.jpg |
| kalanan | Kalanan Riverside Resort | กาลนาน ริเวอร์ไซด์ รีสอร์ท | Nonthaburi | https://www.kalananriverside.com/th/ | venue-images/3-kalanan.jpg |
| rk | RK Riverside Resort & Spa | เรือนเครือวัลย์ ริเวอร์ไซด์ รีสอร์ท แอนด์ สปา | (not stated in PDF) | https://www.rkresortandspa.com/ | venue-images/4-rk.jpg |
| phuwaan | Baan Phu Waan | บ้านผู้หว่าน | Samphran, Nakhon Pathom | https://phuwaan.catholic.or.th/Site/Welcome.html | venue-images/5-phuwaan.jpg |

Notes
- Each image is the full slide from the PDF (photos + Google Maps route from Mahidol University). Re-export higher-resolution crops if needed.
- An earlier PDF also listed **Ruean Phrueksa Boutique Resort**. It is absent from M_E_-_PLACE.pdf, so it was removed. To restore it, add one object to the `VENUES` array and one image.
- Each card also has a Google Maps search link (`https://www.google.com/maps/search/?api=1&query=<name>`).

## 3. Voting rules
- Ranked ballot: choose up to 3 venues; rank 1 is required, ranks 2 and 3 optional; no gaps (no 3rd without 2nd).
- A venue can hold only one rank per ballot; picking a venue for a new rank moves it.
- Scoring (Borda): 1st = 3 pts, 2nd = 2 pts, 3rd = 1 pt. Also show count of 1st-choice votes per venue.
- One ballot per person, identified by a stable user id; the person may edit their ballot (upsert).
- Required field: name. Optional: organization/team, comment.
- Results update live. Everyone sees scores and voter names; only the owner/admin sees the full table and can export CSV.

## 4. Data model (one document/row per voter)
```json
{
  "uid": "stable-user-id",
  "ranks": ["riverton", "neera", "kalanan"],
  "name": "Jane Doe",
  "org": "Team A",
  "comment": "optional",
  "ts": 1790000000000
}
```
Score for venue v = sum over ballots of (3 − index of v in `ranks`) when present.

## 5. UI features
- Thai / English toggle (button top right; remembered per browser; defaults to browser language). All text lives in the `T` object; venue names in `VENUES`.
- Per venue: photo (tap to enlarge), Website link, Map link, location, score bar, 1st-choice count, voter names with rank, three rank buttons (1st/2nd/3rd).
- Form: name, organization, comment, Submit / Update my vote.
- Admin section: voter table + CSV download (UTF-8 with BOM; cells starting with = + - @ are escaped).
- Light/dark mode, mobile-safe layout, all user text inserted with `textContent` (no HTML injection).

## 6. How identity and storage work today (Claude-hosted)
- Uses Claude's built-in runtime: `claude.use('db')`, `claude.use('user')`, `claude.use('downloads')`.
- Rules: everyone can read `votes`; each signed-in person with Contributor access can write only `votes/<own uid>`; owner can write all.
- Limitation: voters must sign in to claude.ai and be shared as Contributor. This is why you may want to host it elsewhere.

## 7. Rebuilding outside Claude
The page logic is plain HTML/CSS/JS. Only four functions touch Claude's runtime and must be replaced:
1. `me()` → returns `{id, name, isOwner}` (or null)
2. `subscribeVotes(cb)` → calls `cb(arrayOfBallots)` on every change
3. `saveVote(ballot)` → upsert by `uid`
4. `downloads.save(...)` → replace with a normal Blob download link

Option A — Supabase (recommended: real DB, free tier, optional login)
```sql
create table votes (
  uid uuid primary key default auth.uid(),
  ranks text[] not null check (array_length(ranks,1) between 1 and 3),
  name text not null,
  org text, comment text,
  ts timestamptz default now()
);
alter table votes enable row level security;
create policy "read all"  on votes for select using (true);
create policy "own write" on votes for insert with check (auth.uid() = uid);
create policy "own edit"  on votes for update using (auth.uid() = uid);
```
```js
import { createClient } from '@supabase/supabase-js';
const sb = createClient(SUPABASE_URL, SUPABASE_ANON_KEY);
// no-login option: sb.auth.signInAnonymously()  (one browser = one ballot)
// stricter option: email magic link / Google sign-in so one person = one ballot
const me = async () => { const { data:{ user } } = await sb.auth.getUser(); return user && { id:user.id, name:user.user_metadata?.name, isOwner:user.email==='ADMIN@EXAMPLE.ORG' } };
const saveVote = b => sb.from('votes').upsert({ uid:b.uid, ranks:b.ranks, name:b.name, org:b.org, comment:b.comment });
const subscribeVotes = cb => {
  const load = async () => cb((await sb.from('votes').select('*')).data.map(r => ({...r, ts:+new Date(r.ts)})));
  load(); sb.channel('votes').on('postgres_changes',{event:'*',schema:'public',table:'votes'},load).subscribe();
};
```
Option B — Google Form / Google Sheets + Apps Script: simplest for non-developers; weaker against duplicate voting unless you require Google sign-in in the form.
Option C — Firebase (Auth + Firestore): same shape as Option A.

Hosting: any static host (Netlify, Vercel, GitHub Pages, Cloudflare Pages). Replace the embedded base64 images with files from `venue-images/` if you want a lighter page.

## 8. Decisions to make before launch
- Who may vote: anyone with the link, or only invited people (affects anonymous vs. sign-in).
- Whether voter names stay public or only the admin sees them.
- Voting deadline and who announces the result.
- Tie-break rule (suggested: most 1st-choice votes, then discussion).

## 9. Test checklist
1. Vote with 1, 2, and 3 ranks; try a gap (3rd without 2nd) and confirm it is rejected.
2. Re-vote and confirm the ballot updates instead of duplicating.
3. Switch TH/EN and confirm all labels, venue names and table headers change.
4. Open on a phone; tap a photo to enlarge; test all Website and Map links.
5. As admin, download the CSV and open it in Excel (Thai text should display correctly).
