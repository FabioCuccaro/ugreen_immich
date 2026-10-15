# Guida passo passo (italiano)

Immich in Docker sul NAS Ugreen, gestito dall'interfaccia web di UGOS: un Google Photos fatto in casa che legge le foto già sincronizzate dall'app UGREEN, senza copiarle, raggiungibile da fuori casa solo con Tailscale. Panoramica e dettagli nel [README](../../README.md) (in inglese). Per portare la libreria di Google Photos: [google-photos.md](google-photos.md).

I nomi dei menu di UGOS e di Immich cambiano con le versioni: se a schermo sono diversi, vale lo schermo (e una PR che corregge la guida è benvenuta).

Nella guida `<IP del NAS>` è l'indirizzo di casa del NAS (per esempio `192.168.1.10`) e `<utente>` è il tuo nome utente di UGOS.

## Prima di iniziare

- **RAM:** Immich chiede almeno 6 GB e ne consiglia 8. Un DXP2800 (Intel N100, 8 GB) lo regge insieme a Plex e qualche altro container.
- **CPU:** la prima elaborazione (miniature, metadati, volti, ricerca, video) tiene il NAS occupato da qualche ora a qualche giorno, secondo quante foto hai. Dopo, ogni foto nuova si elabora in pochi secondi.
- **Database:** Postgres deve stare su un disco del NAS, mai su una cartella di rete. Meglio un SSD NVMe; sugli HDD funziona, solo più lento.
- **Tailscale** sul NAS, per l'accesso da fuori casa: [ugreen_tailscale](https://github.com/FabioCuccaro/ugreen_tailscale).

## 1. Percorso della cartella delle foto

L'app UGREEN sincronizza il telefono nella cartella personale, che in **File** di UGOS di solito compare come *Cartella personale → Photos → MobileBackup*. Dentro Docker la stessa cartella è `/home/<utente>/Photos/MobileBackup`.

Per controllare: in **File**, tasto destro sulla cartella → **Proprietà**, oppure scegli la cartella con il selettore dell'editor del Project.

## 2. Project `immich` in UGOS

1. *Docker → Project → Crea*: nome `immich`, cartella: nella cartella condivisa `docker` crea `immich`.
2. Incolla [`docker-compose.yml`](../../docker-compose.yml) e sostituisci:
   - `<database password>` con una password casuale fatta **solo di lettere e numeri** (`A-Za-z0-9`). Si scrive una volta sola e vale anche per Postgres.
   - `/home/<user>/Photos/MobileBackup` con il percorso del passo 1.
   - `TZ` con il tuo fuso orario.
   - Se `docker/immich` sta su un SSD, cancella la riga `DB_STORAGE_TYPE: 'HDD'`.
3. Avvia. Al primo avvio scarica diversi GB di immagini.
4. Apri `http://<IP del NAS>:2283` e crea l'utente admin.

I dati di Immich vanno in `docker/immich/library` (miniature, video transcodificati, dump del database) e `docker/immich/postgres` (il database).

## 3. Libreria esterna

1. Nell'interfaccia web di Immich: la tua icona in alto a destra → **Amministrazione** → **Librerie esterne** → **Crea libreria**, proprietario: il tuo utente.
2. Nella libreria nuova: **Aggiungi percorso** → `/external/photos`. È il percorso **dentro** il container, non quello del NAS.
3. Se chiede un'espressione cron, è la pianificazione della scansione automatica: `0 * * * *` ricontrolla ogni ora, `0 3 * * *` una volta a notte. Le foto nuove della sincronizzazione UGREEN compaiono dopo la scansione successiva.
4. **Scansiona**. L'avanzamento si vede in **Amministrazione → Lavori**, e la pagina della libreria mostra quante foto e video ha trovato.

Se il conteggio resta a 0: *Docker → Contenitore → `immich_server` → Terminale*, scrivi `ls /external/photos | head`. Se non stampa niente la cartella non è montata: controlla il percorso nel compose.

## 4. Parallelismo dei lavori (N100)

Con 4 core e altri container accanto, valori alti non fanno finire prima: rendono solo lento tutto il NAS. In **Amministrazione → Impostazioni → Impostazioni dei lavori**:

| Lavoro | Valore |
|---|---|
| Transcodifica video | 1 |
| Generazione miniature | 2 |
| Estrazione metadati | 2 |
| Ricerca intelligente | 1 |
| Riconoscimento volti | 1 |
| OCR, se c'è | 1 |
| Libreria, Sidecar, Migrazione, Attività in background | default |

Finita l'elaborazione iniziale questi valori contano poco. Di notte puoi alzare miniature e metadati a 3 o 4; video e machine learning lasciali a 1. Ogni coda si può mettere in pausa dalla pagina Lavori, per esempio mentre guardi Plex.

## 5. App sul telefono

1. Installa l'app **Immich** e accedi con `http://nas:2283` (via Tailscale).
2. **Non attivare il backup dell'app.** Le foto le carica già l'app UGREEN. Nell'app Immich le foto del telefono hanno l'icona "non salvata": è normale, Immich le vede dalla libreria esterna.
3. Facoltativo: *Impostazioni → Rete → Cambio automatico di URL*: il Wi‑Fi di casa con `http://<IP del NAS>:2283` e `http://nas:2283` come indirizzo esterno.
4. Su Android imposta Tailscale come **VPN sempre attiva**, così l'app funziona anche fuori casa.

## 6. Aggiornamenti

Il compose usa il tag `v3`: riscaricando le immagini dal Project in UGOS prendi tutte le versioni di Immich 3, mai Immich 4. **Immich non ha un tag `:stable`**, e `release` passerebbe da solo alla versione principale successiva. Quando esce Immich 4, leggi le note di rilascio e cambia `v3` in `v4` a mano.

## 7. Backup

⚠️ Non ancora provato.

Il RAID protegge da un disco rotto, non da furto, incendio, cancellazioni sbagliate o aggiornamenti andati male. Finché non c'è una copia fuori dal NAS, tieni acceso anche Google Photos (o un altro backup in cloud).

Cosa copiare, per esempio con l'app di backup di UGOS su un disco USB o su un cloud cifrato:

- la cartella delle foto (`/home/<utente>/Photos/MobileBackup`);
- `docker/immich/library`, soprattutto `library/backups`, dove Immich scrive ogni notte un dump del database ([backup e ripristino](https://docs.immich.app/administration/backup-and-restore)).

Non copiare `docker/immich/postgres` mentre il database è acceso: per ripristinare si usa il dump.

## Sicurezza

- Non mettere mai in una repo il compose compilato (password del database), le chiavi API di Immich, le cartelle `library/` e `postgres/`.
- Niente Accesso rapido o UGREENlink, niente porte aperte sul router, niente Tailscale Funnel per Immich: contiene tutta la tua storia fotografica.
