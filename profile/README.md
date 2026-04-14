# 🚑 Gestionale Formazione - CRI Pisa

Sistema digitale per la gestione dei corsi, iscrizioni e verbalizzazione, progettato con un'architettura **Security-first** conforme alle linee guida GDPR per il trattamento dei dati sensibili.

--- 

## 💡 Visione del Progetto
L'obiettivo è centralizzare i flussi della formazione locale, eliminando i processi cartacei e garantendo l'integrità e la riservatezza dei dati dei volontari e dei cittadini.

### Obiettivi chiave:
* **Digitalizzazione:** Gestione completa del ciclo di vita del corso (creazione, iscrizione, esito).
* **Privacy by Design:** Crittografia applicativa sui dati sensibili (CF, anagrafiche).
* **Accessibilità:** Portale responsive per istruttori e segreteria.

---

## 🏗️ Architettura Tecnica e Soluzioni Proposte

### 1. Stack Tecnologico
* **Frontend:** React.js con TypeScript (Hosting: Vercel/AWS).
* **Backend:** Spring Boot 3 (Java 21).
* **Database:** MongoDB Atlas (Tier M0 - Managed Cloud).
* **Security:** HashiCorp Vault (Gestione chiavi di cifratura con AppRole).
* **Storage:** AWS S3 (Archiviazione PDF verbali/attestati).

### 1.1 Architettura di Sistema e Flusso Operativo
<p align="center">
  <img width="600" src="https://github.com/user-attachments/assets/ded0e6de-35ba-4bc7-bae3-925d3ff16e4d" alt="architecture_diagram_cri_pisa">
</p>

L'infrastruttura è stata progettata seguendo il paradigma della **Defense in Depth**, separando nettamente il piano dell'autentificazione, dalla logica applicativa e della persistenza fisica.

#### 1.1.1 Autenticazione e Identity Management (Esterno)
Il flusso inizia all'esterno del perimetro del server (Docker Host) per minimizzare i vettori di attacco:
 * **Delegated Auth:** L'utente interagisce con **Firebase Auth** tramite il frontend React. Questo garantisce che le credenziali (password) non transitino mai sui nostri server.
 * **JWT Tokenomics:** Firebase restituisce un **Bearer Token (JWT)** che il client utilizza per ogni chiamata API. Questo token viene validato dal backend Spring Boot tramite chiavi pubbliche, garantendo l'identità dell'utente senza dover interrogare il database a ogni richiesta.
#### 1.1.2 Logica di Elaborazione e Security Gateway (Backend)
Una volta che la richiesta entra nel Docker Host:
 * **API Gateway:** Funge da Reverse Proxy e punto di ingresso unico, gestendo il routing e terminando la connessione SSL/TLS.
 * **Secret Orchestration (HashiCorp Vault):** SpringBoot non possiede internamente le chiavi di cifratura. Al boot e durante l'esecuzione recupera in modo sicuro la Master Key (AES-256) da Vault. Questo isola le chiavi dal codice e dalle variabili d'ambiente.

#### 1.1.3 Persistenza Stratificata (Data & Object Storage)
L'architettura separa i dati strutturati dagli asset binari per ottimizzare performance, scalabilità e procedure di disaster recovery:

* **Data Persistence (Cloud Managed):** La persistenza dei dati strutturati è affidata a **MongoDB Atlas** (Tier M0). Questa scelta esternalizza la gestione dei backup e dell'alta disponibilità, garantendo che i dati (cifrati a livello applicativo da Spring Boot) siano replicati e sicuri indipendentemente dallo stato del Docker Host.
* **Object Storage (MinIO/S3):** Gli asset binari (PDF di attestati e verbali) vengono archiviati su **AWS S3** (o MinIO in ambiente di staging). Nel database MongoDB viene memorizzato esclusivamente il riferimento (URL o Object ID), prevenendo il "database bloating" e ottimizzando i tempi di risposta delle query.
* **Persistent Volumes (Local Config):** Il Docker Host mantiene volumi persistenti dedicati esclusivamente ai log applicativi e allo stato di **HashiCorp Vault**, garantendo la persistenza delle configurazioni critiche e degli audit log tra i vari riavvii dei container.

Questa architettura garantisce che, anche in caso di compromissione totale del database, i dati sensibili rimangano illeggibili poiché le chiavi sono isolate in un Secret Manager dedicato. La separazione tra Object Storage e Database garantisce inoltre una scalabilità orizzontale semplificata verso il cloud.

### 2. Strategia di Deployment (Proposta)
Ho progettato il sistema per essere scalabile e portabile:
* **Ambiente Locale:** Docker-compose per replicare l'intera infrastruttura (DB, Vault, Storage) con un solo comando.
* **Ambiente Cloud:** Migrazione trasparente verso **AWS** o **DigitalOcean** grazie alla containerizzazione.
* **Costi:** Ottimizzati tramite l'uso di servizi gestiti gratuiti (Atlas M0) e crediti non-profit (TechSoup/AWS).

---

## 📊 Modellazione Dati
Di seguito la struttura delle entità e le relazioni logiche del database.
<p align="center">
  <img width="1000" alt="diagram-export-14-04-2026-21_56_03" src="https://github.com/user-attachments/assets/64d9e2cb-e146-4bcd-b13e-088a3d0c4a71" />
</p>
*Nota: I campi contrassegnati come `_cifrato` vengono gestiti tramite il modulo di crittografia AES-256 integrato con HashiCorp Vault.*

---

## 🔒 Sicurezza e Protezione Dati
Dato il contesto associativo, la sicurezza non è un "add-on" ma il core del sistema:

* **Encryption at Rest:** Implementazione di crittografia AES-256-GCM a livello applicativo per i dati sensibili.
* **Secret Management:** HashiCorp Vault protegge la *Master Key*, impedendo che le chiavi risiedano nel codice o in variabili d'ambiente.
* **Identity Management (Firebase Auth):** Autenticazione delegata a Firebase per garantire la gestione sicura delle password e l'emissione di token **JWT (JSON Web Token)** verificati dal backend Spring Boot.
* **Multi-Factor Authentication (MFA):** Implementazione di un secondo fattore di autenticazione (2FA) tramite standard **TOTP** (libreria Java open-source). Compatibile con Google Authenticator, 1Password e Bitwarden, per proteggere gli account amministrativi.
* **Separazione dei dati:** I file (PDF) risiedono in bucket S3 protetti con policy di accesso dedicate, riducendo il carico sul DB e isolando i file binari dalla logica applicativa.

---

## 🚀 Funzionalità Principali (Features)
Questa sezione descrive i moduli operativi che verranno implementati per soddisfare le esigenze del comitato:
## 1. Modulo Iscrizioni e Utenti
* **Self-Service Iscrizione:** Portale per i volontari per visualizzare i corsi disponibili e iscriversi autonomamente.
* **Gestione Aziende/Enti**: Supporto per l'iscrizione di dipendenti di aziende esterne (formazione legge 81/08).

## 2. Modulo Gestione Corsi
* **Anagrafica Corsi**: Creazione e configurazione di nuovi percorsi formativi (Primo Soccorso, 81/08, BLSD/PBLSD, Manovre salva vita pediatriche e sonno sicuro).
* **Pianificazione** Gestione delle date, delle sedi e assegnazione dei docenti/istruttori.
* **Monitoraggio Status**: Controllo in tempo reale dei corsi (Aperto, In corso, Chiuso, Archiviato).

## 3. Verbalizzazione e Attestati
* **Generazione Automatica PDF:** Creazione dei verbali d'esame e degli attestati di partecipazione con layout ufficiale CRI.
* **QR-Code Verification:** Ogni attestato includerà un QR-Code per la verifica rapida della validità del documento da parte di enti terzi.
* **Notifiche Scadenze:** Sistema di alert automatico per avvisare volontari e segreteria quando un brevetto è in scadenza (Retraining).

## 4. Pannello Amministrativo e Privacy
* **Audit Log:** Visualizzazione dei log di accesso ai dati sensibili per il Responsabile Protezione Dati (DPO).
* **Gestione Ruoli:** Permessi granulari (Admin, Istruttore, Segreteria, Volontario).
