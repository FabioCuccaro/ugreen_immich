# Portare la libreria di Google Photos in Immich

⚠️ **Non ancora provato fino in fondo.** I passi seguono la documentazione di [immich-go](https://github.com/simulot/immich-go) (versione 0.32, la release "Immich V3 compatibility"). Segnalazioni benvenute.

Versione inglese: [google-photos.md](../google-photos.md). Installazione di Immich: [guida](guida.md).

## Perché Takeout + immich-go

Google Takeout esporta data, luogo e descrizione in un file `.json` separato accanto a ogni foto, con nomi non sempre abbinabili (tagli a 47 caratteri, `(1)`, `-modificato`). Spesso la data del file è quella dell'export. Se carichi le foto così come sono, ti ritrovi migliaia di foto datate al giorno dell'export.

immich-go legge direttamente gli zip di Takeout (non serve scompattarli), abbina ogni foto al suo JSON, ripristina data, luogo e descrizione, ricrea gli album e salta quello che è già su Immich (controllo per checksum). Quindi puoi rilanciarlo senza creare doppioni.

## Se il telefono sincronizza già sul NAS

Con la [libreria esterna](guida.md#3-libreria-esterna) attiva, le foto recenti sono già in Immich. Il Takeout serve per i buchi: telefoni precedenti, foto cancellate dal telefono con "Libera spazio", foto salvate dagli album condivisi. Il dry-run del passo 3 ti dice quante ne mancano davvero. Le foto di cui Google ha riscritto i metadati non coincidono per checksum e risultano doppie: si puliscono con **Utilità → Duplicati** in Immich.

## 1. Chiedere il Takeout

Prima di chiederlo, salva nella tua libreria le foto degli album **condivisi da altri** che vuoi tenere: per Takeout non sono tue. Sposta fuori anche quello che hai nella *Cartella protetta*.

Su <https://takeout.google.com>:

1. **Deseleziona tutto** e spunta solo **Google Foto** (lascia inclusi tutti gli album).
2. Formato **.zip**, dimensione massima **50 GB**, così hai meno parti.
3. Google manda un'email quando è pronto (ore o giorni). Scarica **tutte** le parti (`takeout-…-001.zip`, `-002.zip`, …) sul PC e controlla che non ne manchi nessuna e che nessuna sia troncata. I link scadono dopo circa una settimana.

## 2. Chiave API di Immich

In Immich: *Impostazioni account → Chiavi API → Nuova chiave*, con questi permessi:

`asset.read`, `asset.statistics`, `asset.update`, `asset.upload`, `asset.copy`, `asset.delete`, `asset.download`, `album.create`, `album.read`, `albumAsset.create`, `server.about`, `stack.create`, `tag.asset`, `tag.create`, `user.read`, `job.create`, `job.read`.

Tieni la chiave in un password manager, mai in una repo o in una chat.

## 3. Prova a vuoto, poi import vero

Installa immich-go sul PC ([installazione](https://github.com/simulot/immich-go/blob/main/docs/installation.md)) e lancialo da casa, in LAN (via Tailscale funziona, ma è più lento). Da PowerShell su Windows:

```powershell
# prova: legge tutto e fa il resoconto, senza caricare nulla
immich-go upload from-google-photos --dry-run `
  --server=http://<IP del NAS>:2283 --api-key=<chiave API> `
  --include-unmatched --include-partner=false --concurrent-tasks=4 `
  D:\Takeout\takeout-*.zip

# import vero: stesso comando senza --dry-run
```

| Opzione | Default | Perché |
|---|---|---|
| `--sync-albums` | sì | ricrea gli album di Google Photos |
| `--include-archived` | sì | importa anche le foto archiviate |
| `--include-unmatched` | **no** | importa anche i file senza JSON. **Attivala**, altrimenti alcuni file vengono saltati |
| `--include-partner` | sì | importa le foto condivise da un partner. Mettila a `false` se non le vuoi nella tua libreria |
| `--include-trashed` | no | il cestino resta fuori |
| `--people-tag` | sì | i nomi delle persone di Google diventano tag (Google non esporta i riquadri dei volti, Immich li ricalcola) |
| `--pause-immich-jobs` | sì | sospende miniature e ML durante l'upload |
| `--concurrent-tasks` | n. core | 4 basta per non soffocare il NAS |

Alla fine immich-go scrive un resoconto a contatori (trovati, caricati, già presenti, JSON mancanti, errori) e un log con il dettaglio di ogni file.

## 4. Verificare che non manchi niente

1. **Contatori:** immagini e video trovati nel Takeout devono essere uguali a caricati + già presenti. Errori e file scartati vanno letti uno per uno nel log.
2. **Seconda passata:** rilancia lo stesso comando. Deve dire **0 caricati** e tutto già presente.
3. **Per anno:** confronta qualche anno nella timeline di Immich con Google Photos (l'ultimo anno, un anno vecchio, un viaggio con tanti video). Controlla le date delle foto più vecchie, di quelle di WhatsApp e delle scansioni.
4. **Album:** confronta l'elenco degli album e il numero di elementi di 3–4 album grandi.

## Cosa Takeout non porta

- Le foto negli album condivisi da altri che non hai salvato nella tua libreria.
- La *Cartella protetta*, non sempre inclusa.
- La piena qualità, se Google Photos era in "Risparmio spazio": arrivano le copie compresse, perché l'originale su Google non esiste più.
- Le modifiche: l'originale e la versione `-modificato` arrivano come due foto distinte.
- Volti e ricerca: Immich li ricalcola; i nomi vanno riassegnati (i tag delle persone aiutano).

Le foto in movimento del Pixel (`.MP.jpg`) sono supportate e restano "vive".
