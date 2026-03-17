# Guide d'installation — Skill "History Short"

Génère des YouTube Shorts où tu incarnes un personnage historique grâce à l'IA.

---

## Étape 1 : Installer Claude Code

Si ce n'est pas déjà fait :
```bash
npm install -g @anthropic-ai/claude-code
```

Puis lancer : `claude` dans le terminal.

---

## Étape 2 : Installer la skill

```bash
mkdir -p ~/.claude/skills/history-short/
cp history-short-SKILL.md ~/.claude/skills/history-short/SKILL.md
```

Vérifie que ça marche :
```bash
cat ~/.claude/skills/history-short/SKILL.md | head -5
```
Tu dois voir `name: history-short`.

---

## Étape 3 : Installer les dépendances Python

```bash
pip install google-genai requests boto3
```

FFmpeg (si pas déjà installé) :
```bash
# Mac
brew install ffmpeg

# Linux
sudo apt install ffmpeg
```

---

## Étape 4 : Créer tes comptes API

Tu as besoin de 6 services. Voici comment obtenir chaque clé :

### 1. Gemini (génération d'images) — GRATUIT
- Va sur https://aistudio.google.com/apikey
- Clique "Create API Key"
- Copie la clé

### 2. ElevenLabs (voix) — Plan gratuit dispo
- Crée un compte sur https://elevenlabs.io
- Va dans Profile → API Key
- Copie la clé
- **IMPORTANT** : Clone ta voix ! Va dans Voices → Add Voice → Instant Voice Clone
  - Uploade 1-3 enregistrements de ta voix (30s-1min chacun)
  - Note le Voice ID de ta voix clonée

### 3. fal.ai (avatar lip-sync) — $10 de crédits offerts
- Crée un compte sur https://fal.ai
- Va dans Dashboard → Keys → Create Key
- Copie la clé (format : `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`)

### 4. Freepik (animation Kling 3.0) — Plan API requis
- Va sur https://www.freepik.com/api
- Souscris à un plan API
- Copie la clé API

### 5. Suno (musique) — Via sunoapi.org
- Va sur https://sunoapi.org
- Crée un compte et obtiens une clé API

### 6. Cloudflare R2 (stockage fichiers) — GRATUIT (10 Go)
- Crée un compte sur https://dash.cloudflare.com
- Va dans R2 → Create Bucket (nomme-le comme tu veux)
- Va dans R2 → Manage R2 API Tokens → Create API Token
- Note : Account ID, Access Key ID, Secret Access Key
- Active l'accès public sur ton bucket (Settings → Public access → Allow)
- Note l'URL publique du bucket

---

## Étape 5 : Configurer les clés dans Claude Code

Crée ou édite le fichier `~/CLAUDE.md` :

```bash
nano ~/CLAUDE.md
```

Ajoute ce bloc (remplace les `xxx` par tes vraies clés) :

```markdown
# Mes API Keys

| Service | Clé |
|---------|-----|
| Gemini | `AIzaSyXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX` |
| ElevenLabs | `sk_XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX` |
| fal.ai | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx` |
| Freepik | `FPSXxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx` |
| Suno | `xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx` |

# ElevenLabs Voice ID
Ma voix clonée : `XXXXXXXXXXXXXXXXXXXXXX`

# Cloudflare R2
R2_ACCOUNT_ID=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
R2_ACCESS_KEY_ID=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
R2_SECRET_ACCESS_KEY=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
R2_BUCKET_NAME=mon-bucket
R2_PUBLIC_URL=https://pub-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx.r2.dev
```

---

## Étape 6 : Prépare ta photo

Prends un selfie ou une photo de toi :
- Visage bien visible, de face si possible
- Bonne lumière
- Fond pas trop chargé

Mets-la quelque part d'accessible, par exemple :
```bash
cp ma_photo.jpg ~/ma_photo.jpg
```

---

## Utilisation

Lance Claude Code dans ton terminal :
```bash
claude
```

Puis tape :
```
/history-short "Maria Montessori" --photo ~/ma_photo.jpg
```

### Autres exemples :
```
/history-short "Nikola Tesla" --photo ~/ma_photo.jpg
/history-short "Léonard de Vinci" --photo ~/ma_photo.jpg
/history-short "Cléopâtre" --photo ~/ma_photo.jpg
/history-short "Alan Turing" --photo ~/ma_photo.jpg
/history-short "Frida Kahlo" --photo ~/ma_photo.jpg
```

### Options :
```
--scenes 5          # Plus de scènes (défaut: 4)
--musique "jazz"    # Style de musique
--voix ID_VOIX      # Voix ElevenLabs spécifique
```

---

## Ce qui va se passer

1. Claude recherche le personnage (époque, costumes, décors)
2. Il te montre une fiche historique → **tu valides**
3. Il écrit le script → **tu valides**
4. Il génère les images de toi en costume d'époque → **tu valides**
5. Il génère ta voix sur chaque scène → **tu écoutes et valides**
6. Il lance les animations vidéo en parallèle (~5 min)
7. Il monte tout avec musique de fond
8. **La vidéo finale s'ouvre automatiquement !**

Durée totale : ~10-15 min. Coût : ~$4-5 par vidéo.

---

## Dépannage

| Problème | Solution |
|----------|----------|
| `pip install` échoue | Essaie `pip3 install` ou `python3 -m pip install` |
| Freepik 403 "IP blocked" | Attends 10 min et réessaie, ou Claude basculera sur fal.ai |
| fal.ai "Exhausted balance" | Recharge sur https://fal.ai/dashboard/billing |
| ElevenLabs retourne du JSON | Bug temporaire, Claude réessaie automatiquement |
| Avatar lip-sync bizarre | Normal sur les longues scènes. Les scènes avatar doivent faire 7-15s max |
| Voix pas naturelle | Améliore ton clone ElevenLabs avec plus d'échantillons audio |

---

*Créé par Nico Guyon — Formation IA avec Claude Code*
