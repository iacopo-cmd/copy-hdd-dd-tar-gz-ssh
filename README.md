# copy-hdd-dd-tar-gz-ssh — backup di un disco intero via SSH, alla rozza

Copia un disco intero (dd) compresso (gzip/pigz) su un'altra macchina via SSH, a pezzi,
con ripresa automatica. **Niente wget, niente curl, niente git sulla macchina da salvare:**
apri il file giusto qui sopra, copi tutto, incolli nel terminale. Fine.

## Perché così (e perché la chiave è automatica)

Questo strumento serve **quando le cose vanno male**: un crash, un disco che comincia a
dare errori, un server da spostare di corsa, un sistema da salvare prima che muoia.

In quei momenti:
- hai **fretta**;
- sei **agitato**;
- **non hai voglia di scrivere**, e ogni cosa che scrivi è un'occasione per sbagliare
  (una password, un IP, un `dd` col disco sbagliato).

Per questo:
- **si installa con un copia e incolla**: non serve scaricare niente, funziona anche su una
  macchina senza rete verso Internet, basta un terminale;
- **la chiave SSH si fa da sola** (`chiavi.sh`): scrivi la password della destinazione
  **una volta sola**, poi il backup va da solo per ore. Se cade la rete riprova, e dopo un
  riavvio riparte senza chiederti niente;
- **legge tutto da `config.txt`**; se manca o è sbagliato, **ti fa le domande**;
- **si ferma subito se la password o la chiave vengono rifiutate**, invece di insistere
  (insistere = ban di fail2ban proprio quando ti serve entrare).

Meno cose da scrivere = meno errori quando conta.

## Quale file usare

| File | Cosa fa quando lo incolli |
|---|---|
| `tutto-alla-rozza.txt` | **Tutto insieme**: crea `backup-disco.sh`, `config.txt` e `chiavi.sh` in `/root/backup-disco`. Non lancia niente. |
| `alla-rozza.txt` | Solo il backup: `backup-disco.sh` + `config.txt`. |
| `chiavi-alla-rozza.txt` | Solo la chiave: crea `chiavi.sh` **e lo lancia subito**. |
| `key-pezzente.txt` | Solo la chiave, **versione minima**: 40 righe, niente file creati a parte la chiave, si legge a colpo d'occhio. Chiede IP, porta, utente e la password una volta. |

Tutti spengono la history della shell durante l'incolla e la riaccendono alla fine.
Se `config.txt` c'è già **non viene toccato** (il nuovo va in `config.txt.nuovo`).

## In emergenza, passo passo

Sulla macchina **da salvare** (la sorgente), da root:

```text
1. tmux                                  (se cade la tua connessione il backup continua)
2. incolla tutto-alla-rozza.txt
3. /root/backup-disco/chiavi.sh          (IP, porta, utente, password UNA volta)
4. nano /root/backup-disco/config.txt    (DISCO=..., NUMERO=...)
5. /root/backup-disco/backup-disco.sh
```

Si è interrotto? **Rilancia il punto 5 con lo stesso NUMERO**: salta i pezzi già arrivati.

Sulla destinazione trovi:
- `/root/backupN.img.gz` (l'immagine del disco);
- `/root/backupN-info.txt` (disco, tabella partizioni, scelte fatte);
- `/root/backupN-parti/` (i pezzi, da cancellare quando hai verificato).

Ripristino (**sovrascrive tutto il disco di destinazione**):

```text
gunzip -c /root/backupN.img.gz | dd of=/dev/DISCO bs=4M status=progress
```

## PROVALO PRIMA DI AVERNE BISOGNO

Uno strumento d'emergenza vale solo se l'hai già usato a freddo. **Fallo una volta adesso**,
con calma, su una macchina che non ti serve: chiave, backup, ripresa dopo aver staccato la rete,
ripristino. Il giorno del crash non devi scoprire niente di nuovo.

> Stato onesto: gli script sono stati provati per sintassi, controlli del config, domande e
> copia-incolla. **Un giro completo contro un disco e una destinazione veri non è ancora stato
> fatto.** Fallo tu prima che serva.

## Cose da sapere

- **Disco acceso = copia a rischio**: se il sistema (o un database) sta scrivendo, l'immagine
  può non ripartire. Meglio fermare i servizi o avviare da un sistema live.
- **Spazio**: sulla destinazione serve circa il doppio del compresso (pezzi + immagine finale).
- **`gzip -t`** verifica che i file siano integri, non che siano uguali al disco: solo un
  ripristino di prova lo dimostra.
- **Gira da root** e legge dischi interi: controlla due volte `DISCO=` in `config.txt`.
- La chiave (`/root/.ssh/backup-disco_ed25519`) **non ha passphrase**, apposta per andare da sola:
  chi è root sulla sorgente entra sulla destinazione. Finita l'emergenza, se vuoi, toglila da
  `~/.ssh/authorized_keys` sulla destinazione.
- Senza chiave si usa la password con `sshpass` (se installato). Una password scritta a mano
  viene salvata in `.password-<nome>` (chmod 600) fino a backup finito, per sopravvivere ai riavvii.
