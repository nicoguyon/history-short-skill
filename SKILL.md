---
name: history-short
description: "Génère un YouTube Short didactique où l'utilisateur incarne un personnage historique — pipeline complet : recherche image, script, Gemini images, ElevenLabs voix, fal.ai avatar, Kling 3.0 animation, Suno musique, FFmpeg montage"
---

# History Short Generator

Génère un YouTube Short (9:16, 45-60s) où l'utilisateur incarne un personnage historique qui raconte son histoire à la première personne. Style immersif et pédagogique.

## Invocation

```
/history-short "Maria Montessori" --photo ~/ma_photo.jpg
/history-short "Nikola Tesla" --photo ~/ma_photo.jpg --scenes 4
/history-short "Marie Curie" --photo ~/ma_photo.jpg --voix daniel --musique "dramatic orchestral"
```

### Arguments

| Argument | Requis | Description |
|----------|--------|-------------|
| `"Personnage"` | OUI | Nom du personnage historique |
| `--photo` | OUI | Chemin vers la photo de référence de l'utilisateur (visage net, face caméra idéal) |
| `--scenes` | NON | Nombre de scènes (défaut: 4) |
| `--voix` | NON | Voix ElevenLabs : ID ou nom (défaut: demander à l'utilisateur) |
| `--musique` | NON | Style de musique Suno (défaut: adapté à l'époque du personnage) |
| `--ratio` | NON | Format vidéo (défaut: `9:16`) |
| `--output` | NON | Dossier de sortie (défaut: `~/generated_imgs/history-short/{personnage}/`) |

## Prérequis

### API Keys nécessaires

L'utilisateur doit fournir ces clés (dans son CLAUDE.md, en variables d'env, ou quand demandé) :

| Service | Variable | Usage |
|---------|----------|-------|
| **Gemini** | `GEMINI_API_KEY` | Génération d'images (modèle `gemini-3-pro-image-preview`) |
| **ElevenLabs** | `ELEVENLABS_API_KEY` | Voix off et voice clone |
| **fal.ai** | `FAL_KEY` | Avatar lip-sync (Kling Avatar v2 Pro) |
| **Freepik** | `FREEPIK_API_KEY` | Animation Kling 3.0 Pro (image-to-video) |
| **Suno** | `SUNO_API_KEY` | Musique de fond instrumentale |
| **Cloudflare R2** | `R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `R2_BUCKET_NAME`, `R2_PUBLIC_URL` | Hébergement fichiers (URLs publiques pour les APIs) |

### Dépendances système

```bash
pip install google-genai requests boto3
brew install ffmpeg  # ou équivalent
```

### Voix ElevenLabs

L'utilisateur doit avoir au moins une voix configurée. Pour cloner sa voix :
1. Aller sur elevenlabs.io → Voices → Add Voice → Instant Voice Clone
2. Uploader 1-3 samples audio de sa voix
3. Récupérer le Voice ID

Pour lister les voix disponibles :
```bash
curl -s "https://api.elevenlabs.io/v1/voices" -H "xi-api-key: $ELEVENLABS_API_KEY" | python3 -c "import json,sys; [print(f'{v[\"voice_id\"]} - {v[\"name\"]}') for v in json.load(sys.stdin)['voices']]"
```

## ÉTAPE 0 — RECHERCHE HISTORIQUE (CRITIQUE)

**C'est l'étape la plus importante. Sans elle, les images seront anachroniques et la vidéo perdra toute crédibilité.**

Avant TOUTE génération, Claude DOIT rechercher en profondeur :

### Ce qu'il faut rechercher

| Catégorie | Détails à trouver | Exemple (Maria Montessori, 1907) |
|-----------|-------------------|----------------------------------|
| **Époque exacte** | Dates clés, période historique | Italie 1907, Belle Époque, royaume d'Italie |
| **Vêtements homme** | Coupe, tissus, couleurs, accessoires | Costume 3 pièces sombre, col haut blanc, cravate lavallière, chaîne de montre à gousset, chapeau melon ou fedora |
| **Vêtements femme** | Robes, coiffures, accessoires | Robe longue col haut, manches bouffantes, chignon |
| **Architecture** | Style des bâtiments, matériaux | Palais romains, colonnade, terracotta, pierre de taille, cyprès |
| **Intérieurs** | Mobilier, décoration, éclairage | Bois massif, lampes à huile/gaz, sol carrelé ou parquet, murs enduits |
| **Transports** | Véhicules de l'époque | Calèches, premiers tramways, pas d'automobiles courantes |
| **Objets quotidiens** | Outils, instruments, livres | Plumes d'écriture, encriers, cahiers reliés cuir, lunettes rondes |
| **Palette de couleurs** | Tons dominants de l'époque | Tons chauds, sépia, brun, crème, vert olive, bleu marine |
| **Éclairage** | Sources de lumière typiques | Lumière naturelle dominante, bougies, lampes à gaz |

### Comment rechercher

```python
# Utiliser WebSearch pour chaque catégorie :
# "1907 Italy men's fashion clothing"
# "early 1900s Italian school interior"
# "Rome 1900s architecture streets"
# "Montessori original Casa dei Bambini photos"
# "Belle Époque Italian gentleman suit"
```

**Chercher des photos d'époque RÉELLES** (archives, musées) pour s'en inspirer. Pas des reconstitutions modernes.

### Fiche d'identité historique (à remplir avant toute génération)

```
PERSONNAGE : [nom]
ÉPOQUE : [dates précises]
LIEU : [ville, pays]
CONTEXTE : [événement historique]

COSTUME HOMME :
- Veste : [coupe, couleur, tissu]
- Gilet : [style, boutons]
- Chemise : [col, couleur]
- Cravate/nœud : [type]
- Pantalon : [coupe]
- Chaussures : [type]
- Accessoires : [montre, chapeau, lunettes, canne...]
- Coiffure : [style de l'époque]

DÉCORS :
- Extérieur : [architecture, rue, végétation]
- Intérieur 1 : [type de pièce, mobilier, éclairage]
- Intérieur 2 : [...]

OBJETS D'ÉPOQUE : [liste d'objets qui apparaîtront dans les scènes]
PALETTE COULEURS : [tons dominants]
ÉCLAIRAGE : [type de lumière naturelle/artificielle]
```

**Cette fiche doit être montrée à l'utilisateur pour validation AVANT de générer les images.**

## Pipeline complet

```
Personnage historique + Photo utilisateur
    |
    |--[0] RECHERCHE HISTORIQUE : WebSearch pour l'époque, les costumes, l'architecture,
    |      les objets, les couleurs. Remplir la fiche d'identité historique.
    |      >>> VALIDATION UTILISATEUR de la fiche <<<
    |
    |--[1] RECHERCHE VISUELLE : Google Images du personnage et de son époque
    |      → photos d'archives, peintures, reconstitutions
    |      → comprendre le LOOK exact (vêtements, décors, ambiance)
    |
    |--[2] SCRIPT : Claude écrit 4-5 scènes en narration 1ère personne (~50-60s)
    |      - Ton : dramatique, captivant, pédagogique, avec une chute/morale
    |      - Le personnage raconte SON histoire comme s'il était vivant
    |      >>> VALIDATION UTILISATEUR <<<
    |
    |--[3] IMAGES : Gemini Pro génère les images des scènes
    |      - Photo de référence utilisateur + prompt décor/costume d'époque
    |      - Scene 1 et dernière scène : FACE CAMÉRA (pour avatar lip-sync)
    |      - Scènes intermédiaires : angles variés, plans d'action
    |      - Optionnel : 1 scène B-roll sans l'utilisateur (objets, lieux, figurants)
    |      >>> VALIDATION UTILISATEUR <<<
    |
    |--[4] AUDIO : ElevenLabs génère la voix off par scène
    |      - Voix clone de l'utilisateur
    |      - Scene 1 : texte court (7-10s max pour l'avatar)
    |      - Autres scènes : voix off sur les animations
    |      >>> VALIDATION UTILISATEUR (ouvrir les mp3) <<<
    |
    |--[5] UPLOAD R2 : Toutes les images + audios sur URLs publiques
    |
    |--[6] VIDÉOS (en parallèle) :
    |      - Scene 1 → fal.ai Kling Avatar v2 Pro (lip-sync, 7-10s)
    |      - Scènes intermédiaires → Freepik Kling 3.0 Pro (animation cinématique)
    |      - Dernière scène → fal.ai Kling Avatar v2 Pro (lip-sync)
    |
    |--[7] MUSIQUE : Suno génère la musique de fond (en parallèle avec étape 6)
    |
    |--[8] MONTAGE FFmpeg :
    |      - Normaliser toutes les vidéos à 1080x1920
    |      - Ajouter voix off sur les scènes Kling
    |      - Fade in/out audio entre chaque scène (transitions lisses)
    |      - Concat toutes les scènes
    |      - Ajouter musique de fond à 12% de volume
    |      → OUVRIR la vidéo finale
```

## Structure des scènes (modèle par défaut — 4 scènes)

| # | Type | Durée | Contenu | Cadrage |
|---|------|-------|---------|---------|
| 1 | **Avatar** | 7-10s | Introduction : "Je m'appelle X..." | Face caméra, plan moyen |
| 2 | **Kling + VO** | 8-10s | L'invention / le contexte | Action, plan moyen-large |
| 3 | **Kling + VO** | 8-10s | Le moment clé / la découverte | Plan d'action ou B-roll |
| 4 | **Avatar** | 10-15s | La révélation / morale | Gros plan face caméra, intense |

**Total : 45-60 secondes**

### Variante 5 scènes (avec B-roll)

Ajouter une scène B-roll entre les scènes 2 et 3 (figurants, objets, lieux) pour enrichir visuellement. Pas de visage de l'utilisateur dans le B-roll.

## APIs — Détails techniques

### 1. Gemini Pro (génération images)

```python
from google import genai
from google.genai import types

client = genai.Client(api_key=GEMINI_API_KEY)

# Charger la photo de référence
with open(photo_path, "rb") as f:
    ref_data = f.read()
ref_image = types.Part.from_bytes(data=ref_data, mime_type="image/jpeg")

response = client.models.generate_content(
    model="gemini-3-pro-image-preview",
    contents=[ref_image, types.Part.from_text(text=prompt)],
    config=types.GenerateContentConfig(
        response_modalities=["IMAGE", "TEXT"],
        image_config=types.ImageConfig(aspect_ratio="9:16")
    )
)

for part in response.candidates[0].content.parts:
    if part.inline_data:
        with open(output_path, "wb") as f:
            f.write(part.inline_data.data)
```

**Prompts images — règles critiques :**
- TOUJOURS inclure : `"Using the person in the reference photo as the EXACT model (same face, same features, same hair)"`
- Scènes avatar : `"faces the camera directly"`, `"direct eye contact"`
- Décrire le costume d'époque en détail
- Décrire le décor en détail
- Terminer par : `"The face MUST match the reference photo exactly. Photorealistic, cinematic quality."`

### 2. ElevenLabs (voix off)

```python
import requests

resp = requests.post(
    f"https://api.elevenlabs.io/v1/text-to-speech/{voice_id}",
    headers={"xi-api-key": ELEVENLABS_API_KEY, "Content-Type": "application/json"},
    json={
        "text": script_scene,  # AVEC apostrophes françaises : j'ai, c'est, l'invention
        "model_id": "eleven_multilingual_v2",
        "voice_settings": {"stability": 0.5, "similarity_boost": 0.8, "style": 0.3}
    }
)
# Vérifier que c'est bien de l'audio, pas du JSON (erreur)
if resp.headers.get('content-type', '').startswith('audio'):
    with open(output_path, "wb") as f:
        f.write(resp.content)
```

**Piège : toujours vérifier `content-type: audio/mpeg` avant de sauvegarder. Si c'est du JSON, c'est une erreur.**

### 3. Upload R2 (obligatoire avant fal.ai et Kling)

```python
import boto3

s3 = boto3.client('s3',
    endpoint_url=f'https://{R2_ACCOUNT_ID}.r2.cloudflarestorage.com',
    aws_access_key_id=R2_ACCESS_KEY_ID,
    aws_secret_access_key=R2_SECRET_ACCESS_KEY,
    region_name='auto'
)

s3.upload_file(local_path, R2_BUCKET_NAME, f'history-short/{slug}/{filename}',
    ExtraArgs={'ContentType': content_type})
# URL publique : {R2_PUBLIC_URL}/history-short/{slug}/{filename}
```

### 4. fal.ai Kling Avatar v2 Pro (lip-sync)

```python
import requests, time

FAL_HEADERS = {"Authorization": f"Key {FAL_KEY}", "Content-Type": "application/json"}

# Submit
resp = requests.post(
    "https://queue.fal.run/fal-ai/kling-video/ai-avatar/v2/pro",
    headers=FAL_HEADERS,
    json={"image_url": image_url, "audio_url": audio_url}
)
request_id = resp.json()["request_id"]

# Poll (toutes les 25-30s, ~3 min de génération)
while True:
    status_resp = requests.get(
        f"https://queue.fal.run/fal-ai/kling-video/requests/{request_id}/status",
        headers={"Authorization": f"Key {FAL_KEY}"}
    )
    status = status_resp.json().get("status")
    if status == "COMPLETED":
        result = requests.get(
            f"https://queue.fal.run/fal-ai/kling-video/requests/{request_id}",
            headers={"Authorization": f"Key {FAL_KEY}"}
        ).json()
        video_url = result["video"]["url"]
        break
    elif status in ["FAILED", "ERROR"]:
        raise Exception("Avatar generation failed")
    time.sleep(25)

# Télécharger
vid = requests.get(video_url)
with open(output_path, "wb") as f:
    f.write(vid.content)
```

### 5. Freepik Kling 3.0 Pro (animation image-to-video)

```python
FREEPIK_HEADERS = {"x-freepik-api-key": FREEPIK_API_KEY, "Content-Type": "application/json"}

# Submit
resp = requests.post(
    "https://api.freepik.com/v1/ai/video/kling-v3-pro",
    headers=FREEPIK_HEADERS,
    json={
        "start_image_url": image_url,
        "prompt": "Slow cinematic camera movement, gentle parallax, professional documentary style",
        "negative_prompt": "fast movement, shaky, blurry, distorted, face deformation",
        "duration": "10",       # STRING, pas int !
        "aspect_ratio": "9:16",
        "cfg_scale": 0.5
    }
)
task_id = resp.json()["data"]["task_id"]

# Poll
while True:
    resp = requests.get(
        f"https://api.freepik.com/v1/ai/video/kling-v3/{task_id}",
        headers=FREEPIK_HEADERS
    )
    data = resp.json()["data"]
    if data["status"] == "COMPLETED":
        video_url = data["generated"][0]  # ⚠️ C'est generated[0], PAS data.video.url
        break
    elif data["status"] in ["FAILED", "ERROR"]:
        raise Exception("Kling generation failed")
    time.sleep(20)
```

**⚠️ Format réponse Freepik : `data.generated[0]` pour l'URL vidéo (PAS `data.video.url`)**

**⚠️ Si l'IP est bloquée Freepik (403), fallback sur fal.ai :**
```python
# Fallback fal.ai Kling image-to-video
resp = requests.post(
    "https://queue.fal.run/fal-ai/kling-video/v2/master/image-to-video",
    headers=FAL_HEADERS,
    json={"image_url": url, "prompt": prompt, "duration": "10", "aspect_ratio": "9:16"}
)
# Même polling que l'avatar (requests/{id}/status → requests/{id})
```

### 6. Suno (musique de fond)

```python
SUNO_HEADERS = {"Authorization": f"Bearer {SUNO_API_KEY}", "Content-Type": "application/json"}

resp = requests.post("https://api.sunoapi.org/api/v1/generate",
    headers=SUNO_HEADERS,
    json={
        "prompt": "Warm Italian cinematic orchestral, gentle piano, nostalgic, 80 BPM, 60 seconds",
        "customMode": False,
        "instrumental": True,
        "model": "V4_5ALL",
        "callBackUrl": "https://example.com/cb"  # requis même si inutile
    })
task_id = resp.json()["data"]["taskId"]

# Poll
while True:
    resp = requests.get(
        f"https://api.sunoapi.org/api/v1/generate/record-info?taskId={task_id}",
        headers=SUNO_HEADERS
    )
    data = resp.json()["data"]
    if data["status"] == "SUCCESS":
        audio_url = data["response"]["sunoData"][0]["audioUrl"]
        break
    time.sleep(15)
```

**Adapter le prompt musique à l'époque du personnage :**
- Antiquité → "epic ancient Mediterranean, lyra, percussion"
- Moyen-Âge → "medieval orchestral, lute, choir"
- Renaissance → "Renaissance Italian, harpsichord, strings"
- XVIIIe → "classical orchestral, Mozart era, elegant"
- XIXe → "romantic orchestral, Chopin era, emotional piano"
- XXe → "cinematic orchestral, dramatic strings, documentary"

### 7. FFmpeg (montage final)

**Étape par étape :**

```bash
# 1. Vérifier les specs de chaque vidéo
for f in *.mp4; do echo "=== $f ===" && ffprobe -v error -show_entries stream=codec_type,width,height,channels,sample_rate -of csv=p=0 "$f"; done

# 2. Normaliser chaque part à 1080x1920 + audio stéréo 44100Hz
# Pour les avatars (déjà avec audio) :
ffmpeg -y -i avatar.mp4 \
  -vf "scale=1080:1920:force_original_aspect_ratio=decrease,pad=1080:1920:(ow-iw)/2:(oh-ih)/2" \
  -c:v libx264 -preset fast -crf 18 -r 24 -c:a aac -ac 2 -ar 44100 -b:a 192k \
  part_avatar.mp4

# Pour les Kling (sans audio, ajouter voix off) :
ffmpeg -y -i kling.mp4 -i voiceover.mp3 \
  -vf "scale=1080:1920:force_original_aspect_ratio=decrease,pad=1080:1920:(ow-iw)/2:(oh-ih)/2" \
  -c:v libx264 -preset fast -crf 18 -r 24 \
  -map 0:v -map 1:a -c:a aac -ac 2 -ar 44100 -b:a 192k -shortest \
  part_kling.mp4

# 3. Ajouter des fades audio pour des transitions lisses entre scènes voix off
# Scene avatar d'intro : fade out à la fin
ffmpeg -y -i part.mp4 -t DURATION -af "afade=t=out:st=FADE_START:d=0.7" ...

# Scènes voix off intermédiaires : fade in + fade out
ffmpeg -y -i part.mp4 -af "afade=t=in:st=0:d=0.5,afade=t=out:st=FADE_START:d=1" ...

# 4. Concat
printf "file 'part1.mp4'\nfile 'part2.mp4'\n..." > concat.txt
ffmpeg -y -f concat -safe 0 -i concat.txt \
  -c:v libx264 -preset fast -crf 18 -r 24 -ac 2 -ar 44100 -b:a 192k \
  raw.mp4

# 5. Ajouter musique de fond à 12%
DURATION=$(ffprobe -v error -show_entries format=duration -of csv=p=0 raw.mp4 | cut -d. -f1)
FADE_OUT=$((DURATION - 2))
ffmpeg -y -i raw.mp4 -i music.mp3 \
  -filter_complex "[1:a]atrim=0:${DURATION},afade=t=in:st=0:d=2,afade=t=out:st=${FADE_OUT}:d=2,volume=0.12[music];[0:a][music]amix=inputs=2:duration=first[aout]" \
  -map 0:v -map "[aout]" -c:v libx264 -preset fast -crf 18 -c:a aac -b:a 192k -shortest \
  final.mp4
```

**Règles FFmpeg critiques :**
- TOUJOURS normaliser audio (channels + sample_rate) AVANT concat — mono vs stéréo = artefacts
- TOUJOURS re-encoder le final complet (`-c:v libx264 -c:a aac`), JAMAIS `-c copy`
- Volume musique : `0.12` (12%) — la voix doit dominer
- Scene avatar intro : couper au bon moment, avant les mots qui butent
- Fade audio entre les scènes voix off pour des transitions propres

## Exemple de script (Maria Montessori)

```
SCENE 1 (Avatar, 7-10s) :
"Je m'appelle Claudio Montessori. Et ma sœur Maria... elle a révolutionné l'éducation."

SCENE 2 (Kling + VO, 8-10s) :
"Son idée était simple mais radicale : laisser les enfants choisir leurs activités. Ces petits cubes, ces lettres en bois, ce ne sont pas des jouets. Ce sont des outils de liberté."

SCENE 2b - B-roll enfants (Kling + VO, 8-10s) :
"Regardez-les. Personne ne leur dit quoi faire. Ils choisissent, ils explorent, ils se concentrent. Maria appelait ça le travail de l'enfant."

SCENE 3 (Kling + VO, 8-10s) :
"Maria passait des heures à observer. Elle notait tout dans son carnet. Et ce qu'elle a découvert a changé le monde : un enfant qu'on respecte devient un adulte qui se respecte."

SCENE 4 (Avatar, 10-15s) :
"Aujourd'hui, la méthode Montessori est dans 140 pays. Google, Amazon, Wikipedia... leurs fondateurs sont tous passés par des écoles Montessori. Ma sœur avait raison. Il faut parfois faire confiance aux plus petits, pour voir les plus grandes choses."
```

## Conseils pour le script

### RÈGLE #1 : LE SPECTATEUR DOIT APPRENDRE QUELQUE CHOSE

Chaque vidéo DOIT contenir **au moins 3 faits concrets** que le spectateur ne connaissait probablement pas. C'est ce qui fait que les gens partagent la vidéo. Exemples :
- Un chiffre surprenant : "En 1907, 90% des enfants de ce quartier de Rome ne savaient pas lire"
- Un fait contre-intuitif : "Maria était médecin, pas enseignante — la première femme médecin d'Italie"
- Un lien inattendu avec le présent : "Les fondateurs de Google, Amazon et Wikipedia sont tous passés par Montessori"
- Une anecdote méconnue : "Le premier jour, les enfants ont rangé le matériel tout seuls — personne ne leur avait demandé"

### Structure narrative

1. **Hook immédiat** (scene 1) : "Je m'appelle X..." + fait marquant surprenant
2. **Le contexte** (scene 2) : L'invention/découverte expliquée simplement — ÉDUQUER
3. **Le moment clé** (scene 3) : L'anecdote méconnue qui rend l'histoire vivante — SURPRENDRE
4. **La chute** (scene 4) : Lien avec le monde d'aujourd'hui + morale universelle — INSPIRER

### Règles d'écriture

- **Première personne** : "Je m'appelle...", "Ma sœur...", "J'ai inventé..."
- **Anecdotes concrètes** : détails visuels, chiffres précis, dates, noms de lieux
- **Vulgarisation** : expliquer les concepts complexes comme si on parlait à un ami
- **Humour léger** : une touche d'humour rend le personnage attachant
- **Personnage frère/sœur/ami** : Si le personnage est une femme et l'utilisateur un homme (ou vice-versa), incarner un proche (frère, assistant, collaborateur)
- **PAS de jargon académique** : parler comme un humain, pas comme un manuel d'histoire

## Coût estimé par vidéo

| Service | Coût |
|---------|------|
| Gemini Pro (4-5 images) | ~$1.00 |
| ElevenLabs (5 audios) | ~$0.50 |
| fal.ai Avatar Pro (2 scènes) | ~$2.50 |
| Freepik Kling 3.0 (2-3 scènes) | ~$0.50 |
| Suno musique | ~$0.10 |
| **Total** | **~$4-5 par vidéo** |

## Instructions pour Claude

### RÈGLE ABSOLUE : chaque étape avec "VALIDATION" doit être montrée à l'utilisateur AVANT de passer à la suivante.

### Ordre d'exécution
1. Rechercher le personnage (contexte historique, époque, style vestimentaire)
2. Écrire le script → **MONTRER et VALIDER**
3. Générer les images avec Gemini → **MONTRER et VALIDER** (ouvrir les images)
4. Générer les audios → **OUVRIR les mp3 pour validation**
5. Uploader sur R2
6. Lancer EN PARALLÈLE : avatars fal.ai + animations Kling/Freepik + musique Suno
7. Montage FFmpeg → **OUVRIR la vidéo finale**
8. Itérer sur les ajustements (point de coupe, transitions, volume)

### Parallélisation
- Étape 6 : lancer tous les jobs vidéo + musique en parallèle (background)
- Utiliser `run_in_background=true` pour les polls
- Ne PAS attendre un job pour en lancer un autre

### Gestion des erreurs
- fal.ai balance épuisée → informer l'utilisateur, proposer Ken Burns FFmpeg en fallback
- Freepik IP bloquée → fallback sur fal.ai Kling image-to-video
- ElevenLabs retourne du JSON au lieu d'audio → re-tenter, vérifier content-type
- Kling timeout → re-poll avec les mêmes request_id (les jobs continuent côté serveur)
