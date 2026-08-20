# Sincronizzazione multi-dispositivo (telefono ⇄ PC) con Supabase

L'app continua a salvare in locale su ogni dispositivo (funziona anche offline). In
più, se configuri Supabase e fai login, allinea automaticamente **spese e foto** tra
telefono e PC: aggiungi uno scontrino sul telefono, apri l'app sul PC con lo stesso
account, e lo ritrovi. La sincronizzazione parte da sola all'avvio dell'app e ogni
volta che la riporti in primo piano; c'è anche il pulsante **↻ Sincronizza ora**.

I dati sono privati: protetti dal tuo login, solo tu li vedi.

## Configurazione (una volta sola, ~10 minuti)

### 1. Crea il progetto Supabase (gratis)
1. Vai su supabase.com → **Start your project** → accedi (con GitHub o email).
2. **New project**. Dai un nome (es. `notaspese`), scegli una **Region** europea (es.
   Frankfurt), imposta una **Database password** (annotala, serve solo al progetto) e
   crea. Attendi ~2 minuti che il progetto sia pronto.

### 2. Crea tabelle, regole e approvazione
Nel progetto, apri **SQL Editor** (icona a sinistra) → **New query**, incolla **tutto**
questo e premi **Run**:

```sql
-- Tabella delle spese
create table if not exists public.spese (
  id text primary key,
  user_id uuid not null default auth.uid(),
  data text, tipo text, importo numeric, cliente text, comune text,
  persone int, carta text, az numeric, note text, foto text,
  created bigint, updated bigint, deleted boolean default false
);
alter table public.spese enable row level security;

-- Tabella profili con flag di APPROVAZIONE
create table if not exists public.profiles (
  id uuid primary key references auth.users(id) on delete cascade,
  email text,
  approved boolean not null default false,
  created_at timestamptz default now()
);
alter table public.profiles enable row level security;

-- Ognuno vede/inserisce solo il proprio profilo (l'approvazione la fai tu da dashboard)
drop policy if exists "profili: leggi il proprio" on public.profiles;
create policy "profili: leggi il proprio" on public.profiles
  for select to authenticated using (auth.uid() = id);
drop policy if exists "profili: crea il proprio" on public.profiles;
create policy "profili: crea il proprio" on public.profiles
  for insert to authenticated with check (auth.uid() = id and approved = false);

-- Alla registrazione crea automaticamente un profilo NON approvato
create or replace function public.handle_new_user()
returns trigger language plpgsql security definer set search_path = public as $$
begin
  insert into public.profiles(id,email,approved) values (new.id,new.email,false)
  on conflict (id) do nothing;
  return new;
end; $$;
drop trigger if exists on_auth_user_created on auth.users;
create trigger on_auth_user_created after insert on auth.users
  for each row execute function public.handle_new_user();

-- RECUPERO: crea la riga profilo per gli utenti già registrati che non ce l'hanno
insert into public.profiles (id, email, approved)
select u.id, u.email, false
from auth.users u
left join public.profiles p on p.id = u.id
where p.id is null;

-- Accesso ai dati SOLO se l'utente è approvato
drop policy if exists "solo i propri dati" on public.spese;
create policy "solo i propri dati" on public.spese
  for all to authenticated
  using (auth.uid() = user_id and exists (select 1 from public.profiles p where p.id = auth.uid() and p.approved))
  with check (auth.uid() = user_id and exists (select 1 from public.profiles p where p.id = auth.uid() and p.approved));
```

### 3. Impostazioni di registrazione
- **Authentication → Sign In / Providers → Email** → disattiva **Confirm email** →
  Save (così l'unico filtro è la tua approvazione, non serve anche la conferma mail).
- Se vuoi impedire del tutto le auto-registrazioni via link diretto puoi lasciare
  attivo "Allow new users to sign up": tanto, senza la tua approvazione, un nuovo
  iscritto **non può comunque accedere ai dati**.

### 3-bis. Approvare una registrazione (il tuo compito da amministratore)
Quando qualcuno si registra dall'app, gli arriva il messaggio "in attesa di
approvazione". Per approvarlo:
1. Vai su **Table Editor → profiles**.
2. Trova la riga con la sua email e metti la spunta/valore **approved = true** → salva.
3. Da quel momento quell'utente può accedere. Per revocare l'accesso, rimetti
   `approved = false`.

### 3-ter. Se un utente registrato NON compare in `profiles`
Vuol dire che il trigger non era attivo quando si è registrato. Rimedio:
1. Riesegui il blocco SQL del punto 2 (è idempotente: puoi rieseguirlo senza danni).
   L'ultima query "RECUPERO" crea le righe mancanti per gli utenti già esistenti.
2. Verifica che il trigger esista, con questa query:
   ```sql
   select tgname from pg_trigger where tgname = 'on_auth_user_created';
   ```
   Deve restituire una riga. Se non la restituisce, ripeti il punto 2.
3. Controlla che ora le righe ci siano: `select id, email, approved from public.profiles;`

### 4. La connessione è già incorporata nell'app
Non serve più incollare URL e chiave da nessuna parte: la connessione al progetto
Supabase è **incorporata direttamente nell'app** (usa la *anon key* pubblica, sicura da
distribuire perché i dati restano protetti dalle regole RLS + dalla tua approvazione).
Chi apre l'app deve solo fare **login** o inviare una **richiesta di registrazione**.

> Se un giorno cambi progetto Supabase, aggiorna le costanti `SUPA_URL` e `SUPA_KEY`
> in cima allo script di `index.html` e ricarica la pagina su GitHub.

### 5. Primo accesso (tu, amministratore)
1. Apri l'app (`https://jaycee09.github.io/notaspese/`): comparirà la schermata di login.
2. Tocca **Registrati**, inserisci email e password → **Invia richiesta di
   registrazione**. Vedrai "Richiesta inviata! Potrai accedere dopo l'approvazione".
3. Approva te stesso: **Table Editor → profiles**, trova la tua riga e imposta
   **approved = true** → salva. (Vedi anche il punto 3-bis.)
4. Torna nell'app e fai **Accedi**: entri nell'app e la sincronizzazione parte da sola.

### 6. Nuovi utenti (colleghi) e altri dispositivi
- **Nuovo collega:** gli basta aprire lo stesso indirizzo dell'app, toccare
  **Registrati** e inviare la richiesta. Poi **tu** lo approvi da **Table Editor →
  profiles** (`approved = true`). Da quel momento accede. Nessun link di invito da
  generare o inviare.
- **Altro dispositivo tuo (es. il PC):** apri lo stesso indirizzo e fai **Accedi** con
  la **stessa email e password**. Ritrovi le stesse spese.

## Evitare la pausa del progetto (keep-alive automatico)
I progetti Supabase gratuiti vanno in **pausa dopo 7 giorni consecutivi senza nessuna
richiesta al database** (i dati non si perdono, ma il progetto resta irraggiungibile
finché non lo riattivi a mano). Per evitarlo del tutto c'è un keep-alive automatico via
**GitHub Actions**, che fa una micro-richiesta al database due volte a settimana.

Installazione (una volta sola):
1. Nel repository GitHub dell'app, crea il file `.github/workflows/keepalive.yml`
   (da **Add file → Create new file**: digita il percorso con le `/`, incolla il
   contenuto fornito, fai commit).
2. Vai nella tab **Actions**: comparirà "Supabase keep-alive". Puoi lanciarlo subito con
   **Run workflow** per verificare (esito verde = ok; nei log HTTP 200 oppure 401/403
   vanno bene lo stesso, la richiesta ha comunque raggiunto il database).
3. Da qui in poi parte da solo (lunedì e giovedì) e il progetto non va più in pausa.

Nota: se il repository resta **fermo ~60 giorni** (nessun commit), GitHub sospende i
workflow schedulati. Se un giorno vedi il keep-alive disattivato, riabilitalo con un
clic dalla tab Actions. La chiave usata nel workflow è la anon pubblica (la stessa già
nell'app), quindi è sicuro lasciarla lì.

Alternative al keep-alive: usare l'app almeno una volta a settimana (ogni login/sync è
già una richiesta valida), oppure un servizio di monitoraggio gratuito (cron-job.org,
UptimeRobot) che pinga lo stesso indirizzo REST ogni pochi giorni.

## Note utili
- **Prima sincronizzazione:** dopo il login premi una volta ↻ Sincronizza ora per
  essere sicuro che tutto sia allineato.
- **Foto incluse:** vengono sincronizzate anche le immagini degli scontrini. Il consumo
  dati è contenuto (poche spese al mese), ma la prima sincronizzazione di un mese pieno
  può scaricare qualche MB — meglio in wi-fi.
- **Conflitti:** se modifichi la stessa spesa su due dispositivi, vince la versione
  salvata più di recente.
- **Progetto in pausa:** con il keep-alive attivo (vedi sezione sopra) non dovrebbe più
  capitare. Se comunque un giorno la sync non parte perché il progetto è in pausa, apri
  la dashboard di Supabase e premi **Restore/Resume**: bastano pochi secondi, i dati
  sono intatti.
- **Sicurezza:** la "anon key" è pensata per stare nell'app; i dati restano protetti
  perché la tabella ha le regole (RLS) che mostrano a ciascuno solo le proprie righe.
