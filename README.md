"""
💥 Comic Craft - AI Comic Story Creator (Single-File Standalone App)
Run with: streamlit run app.py
"""

import streamlit as st
import io
import json
import random
import os
import urllib.parse
import requests
import base64
import textwrap
from typing import Dict, Any, List, Optional
from PIL import Image, ImageDraw, ImageFont

# Optional ReportLab for PDF export
try:
    from reportlab.lib.pagesizes import letter
    from reportlab.platypus import SimpleDocTemplate, Paragraph, Image as RLImage, Table
    from reportlab.lib.styles import getSampleStyleSheet, ParagraphStyle
    HAS_REPORTLAB = True
except ImportError:
    HAS_REPORTLAB = False

# Optional Google GenAI
try:
    from google import genai
    from google.genai import types
    HAS_GENAI = True
except ImportError:
    HAS_GENAI = False

# -----------------------------------------------------------------------------
# 1. STREAMLIT CONFIG & CUSTOM COMIC STYLING
# -----------------------------------------------------------------------------
st.set_page_config(
    page_title="Comic Craft - AI Comic Story Creator",
    page_icon="💥",
    layout="wide",
    initial_sidebar_state="expanded"
)

COMIC_CSS = """
<style>
@import url('https://fonts.googleapis.com/css2?family=Bangers&family=Comic+Neue:ital,wght@0,400;0,700;1,700&family=Montserrat:wght@500;700;900&display=swap');

/* Comic Book Global Header */
.comic-title-banner {
    background: linear-gradient(135deg, #e52d27 0%, #b31217 100%);
    padding: 22px;
    border-radius: 12px;
    border: 4px solid #111;
    box-shadow: 6px 6px 0px #111;
    text-align: center;
    margin-bottom: 20px;
}

.comic-title-banner h1 {
    font-family: 'Bangers', cursive !important;
    font-size: 3.2rem !important;
    letter-spacing: 3px;
    color: #fffb00 !important;
    text-shadow: 3px 3px 0px #000, -2px -2px 0px #000, 2px -2px 0px #000, -2px 2px 0px #000, 4px 4px 0px rgba(0,0,0,0.4);
    margin: 0;
    line-height: 1.1;
}

.comic-title-banner p {
    font-family: 'Comic Neue', cursive, sans-serif !important;
    font-weight: 700;
    color: #ffffff !important;
    font-size: 1.15rem;
    margin-top: 6px;
    text-shadow: 2px 2px 0px #000;
}

/* Story Narrative Box */
.story-script-container {
    background: #fffdf7;
    border: 3.5px solid #111;
    border-radius: 10px;
    box-shadow: 6px 6px 0px #111;
    padding: 20px 24px;
    margin-bottom: 24px;
    font-family: 'Comic Neue', cursive, sans-serif;
}

.story-script-container h3 {
    font-family: 'Bangers', cursive;
    color: #d81b60;
    letter-spacing: 1px;
    margin-top: 0;
    font-size: 1.8rem;
}

.narrative-paragraph {
    font-size: 1.1rem;
    line-height: 1.65;
    color: #1a1a1a;
    margin-bottom: 8px;
}

/* Comic Panel Card */
.comic-panel-box {
    background: #ffffff;
    border: 4px solid #111111;
    border-radius: 10px;
    box-shadow: 7px 7px 0px #111111;
    margin-bottom: 22px;
    overflow: hidden;
    display: flex;
    flex-direction: column;
}

.panel-header-strip {
    background-color: #111111;
    color: #ffeb3b;
    padding: 8px 14px;
    font-family: 'Bangers', cursive;
    font-size: 1.25rem;
    letter-spacing: 1.5px;
    display: flex;
    justify-content: space-between;
    align-items: center;
}

.panel-image-container {
    width: 100%;
    position: relative;
    background: #23272e;
    min-height: 260px;
    display: flex;
    justify-content: center;
    align-items: center;
}

.panel-image-container img {
    width: 100%;
    height: auto;
    display: block;
    object-fit: cover;
}

.image-placeholder {
    padding: 30px 20px;
    text-align: center;
    color: #ced4da;
    font-family: 'Comic Neue', cursive, sans-serif;
    font-size: 1.05rem;
    font-weight: 700;
}

/* Comic Narration Box */
.comic-caption-box {
    background: #ffeb3b;
    color: #111111;
    border: 2.5px solid #111111;
    font-family: 'Comic Neue', cursive, sans-serif;
    font-weight: 700;
    font-size: 0.95rem;
    padding: 8px 12px;
    margin: 10px 10px 6px 10px;
    box-shadow: 3px 3px 0px #111111;
    text-transform: uppercase;
    line-height: 1.35;
}

/* Comic Speech Bubble */
.conversation-thread {
    padding: 8px 10px 12px 10px;
    display: flex;
    flex-direction: column;
    gap: 8px;
}

.comic-speech-bubble {
    position: relative;
    background: #ffffff;
    border: 2.5px solid #111111;
    border-radius: 16px;
    padding: 9px 13px;
    font-family: 'Comic Neue', cursive, sans-serif;
    font-weight: 700;
    font-size: 0.98rem;
    color: #111111;
    line-height: 1.35;
    box-shadow: 2px 2px 0px rgba(0,0,0,0.18);
}

.speaker-hero {
    border-left: 6px solid #0052cc;
    background: #f7faff;
}

.speaker-villain {
    border-left: 6px solid #d32f2f;
    background: #fff8f8;
}

.comic-speech-bubble .speaker-name {
    font-family: 'Bangers', cursive;
    font-size: 1.05rem;
    letter-spacing: 0.5px;
    margin-right: 6px;
}

.speaker-hero .speaker-name {
    color: #0052cc;
}

.speaker-villain .speaker-name {
    color: #d32f2f;
}

.comic-thought-bubble {
    border-radius: 22px;
    border-style: dashed;
    background: #f3f6ff;
}

.comic-shout-bubble {
    background: #fffde7;
    border: 3px solid #d32f2f;
    font-weight: 800;
}

/* Action Sound Effect (SFX) */
.comic-sfx-badge {
    display: inline-block;
    background: #ff3d00;
    color: #fffb00;
    font-family: 'Bangers', cursive;
    font-size: 1.35rem;
    letter-spacing: 2px;
    padding: 4px 12px;
    border: 3px solid #111111;
    border-radius: 4px;
    box-shadow: 3px 3px 0px #111111;
    transform: rotate(-4deg);
    text-shadow: 2px 2px 0px #000;
    margin: 6px 10px 0 10px;
    align-self: flex-end;
}
</style>
"""

st.html(COMIC_CSS)

# -----------------------------------------------------------------------------
# 2. GENRES, STYLES & RICH CONVERSATIONAL PRESETS
# -----------------------------------------------------------------------------
GENRES = {
    "Cyberpunk / Sci-Fi": {
        "icon": "🤖",
        "sfx": ["BZZZZT!", "GLITCH!", "CLACK-PUMP!", "VROOOM!", "PING!"],
        "captions": ["Night City never sleeps, it only reboots...", "The neural link flashed critical red...", "Sub-level 44 was off-limits.", "Transmission finalized."]
    },
    "Superhero": {
        "icon": "⚡",
        "sfx": ["BOOOM!", "KRAA-SH!", "ZAPPP!", "THWACK!", "KRAKOOM!"],
        "captions": ["Meanwhile, across the skyline...", "Against impossible odds...", "With seconds left on the clock...", "Justice never sleeps."]
    },
    "Manga / Shonen": {
        "icon": "⚔️",
        "sfx": ["DODODO!", "SHING!", "BA-DUMP!", "FWOOSH!", "GOGOGO!"],
        "captions": ["Deep within, a dormant power stirred...", "The moment of truth had arrived.", "This was far from their final form.", "A new legend was born."]
    },
    "Noir Detective": {
        "icon": "🕵️",
        "sfx": ["BANG!", "CLICK!", "SKRRRRT!", "THUD!", "CREAAAK..."],
        "captions": ["The rain washed the streets, but not the sins...", "Trouble walked in wearing a trenchcoat...", "Midnight came and went without mercy.", "Another cold case in a colder city."]
    },
    "Fantasy / D&D": {
        "icon": "🐉",
        "sfx": ["ROAAAAR!", "SHWING!", "CRACKLE!", "THUMP!", "SWOOSH!"],
        "captions": ["Deep in the Whispering Crypt...", "The runes ignited with eldritch flame...", "The prophecy had warned them.", "The kingdom's dawn broke at last."]
    },
    "Comedy / Slice of Life": {
        "icon": "☕",
        "sfx": ["GASP!", "PLOP!", "BONK!", "SPLAT!", "WHEEEZE!"],
        "captions": ["Everything was completely under control. It wasn't.", "Monday morning hit differently.", "It was at this exact moment they realized...", "And that was only round one."]
    }
}

ART_STYLES = {
    "Classic Western Comic (Jack Kirby / Stan Lee)": "vintage 1970s comic book style, bold black ink outlines, dynamic poses, Ben-Day dots, vibrant primary colors, dramatic shadows, authentic comic print texture, retro Marvel style",
    "Japanese Manga (Screentone B&W)": "authentic Japanese manga style, monochrome black and white ink, screentone shading, dynamic speed lines, intense expressive eyes, shonen jump aesthetic, sharp pen crosshatching",
    "Dark Graphic Novel (Frank Miller / Sin City)": "dark noir graphic novel art, high contrast chiaroscuro, heavy black ink shadows, stark splash of single accent color, gritty urban comic aesthetic",
    "Modern Comic Book (Digital Vibrant)": "modern 2020s digital comic book illustration, crisp vector ink lines, cinematic volumetric lighting, rich color grading, Marvel modern cinematic art style",
    "Saturday Morning Cartoon (90s Animation)": "90s Saturday morning action cartoon style, cel shaded, clean bold outlines, vibrant saturated palette, expressive energetic character art",
    "Watercolor Indie Graphic Novel": "indie graphic novel art, soft hand-painted watercolor textures, organic expressive pencil line art, atmospheric, storybook comic illustration"
}

CAMERA_ANGLES = [
    "Wide Establishing Shot",
    "Medium Shot",
    "Close-up (Face/Emotion)",
    "Extreme Close-up (Eyes/Detail)",
    "Low Angle (Heroic & Imposing)",
    "High Angle / Bird's Eye View",
    "Dutch Angle (Tilted & Tense)"
]

BUBBLE_TYPES = ["Speech", "Shout / Scream", "Thought Bubble", "Whisper / Mumble"]

PRESET_STORIES = [
    {
        "title": "The Awakening of Unit Zero",
        "genre": "Cyberpunk / Sci-Fi",
        "style": "Modern Comic Book (Digital Vibrant)",
        "premise": "A lone cybernetic scavenger discovers a submerged laboratory and accidentally wakes an ancient AI defense automaton.",
        "protagonist": "Kaelen, cybernetic scavenger with glowing orange visor and tactical duster",
        "antagonist": "Unit Zero, towering dormant guardian robot with crimson optics",
        "panels": 4
    },
    {
        "title": "Quantum Coffee Meltdown",
        "genre": "Comedy / Slice of Life",
        "style": "Modern Comic Book (Digital Vibrant)",
        "premise": "A tired college barista invents an espresso machine that triggers a 5-minute time loop every time an impatient CEO demands extra foam.",
        "protagonist": "Maya, overworked barista in a neon green apron with unruly curls",
        "antagonist": "Mr. Grim, impatient corporate executive checking his gold watch",
        "panels": 4
    },
    {
        "title": "The Crimson Thunderbolt Returns",
        "genre": "Superhero",
        "style": "Classic Western Comic (Jack Kirby / Stan Lee)",
        "premise": "When a 50-foot steam-powered war colossus marches toward city hall, a retired veteran hero powers up his ancient plasma gauntlets.",
        "protagonist": "Captain Thunderbolt, weathered silver-haired hero in a gold & navy retro suit",
        "antagonist": "Mecha-Colossus, steam-powered war engine firing plasma bolts",
        "panels": 4
    },
    {
        "title": "Rain on 5th Street",
        "genre": "Noir Detective",
        "style": "Dark Graphic Novel (Frank Miller / Sin City)",
        "premise": "A private eye is hired to investigate a diamond heist scheduled to happen tomorrow night, only to find the client is the mastermind.",
        "protagonist": "Jack Steele, gruff private investigator in a trench coat and fedora",
        "antagonist": "The Broker, mysterious silhouette in an alleyway holding a ledger",
        "panels": 4
    }
]

# -----------------------------------------------------------------------------
# 3. RICH STORY & MULTI-TURN CONVERSATION ENGINE
# -----------------------------------------------------------------------------
def generate_rich_story_procedural(
    title: str,
    genre: str,
    art_style: str,
    premise: str,
    protagonist: str,
    antagonist: str,
    num_panels: int = 4
) -> Dict[str, Any]:
    """Generates continuous story prose and back-and-forth character dialogue."""
    protag_name = protagonist.split(",")[0].strip() or "Hero"
    antag_name = antagonist.split(",")[0].strip() or "Nemesis"

    if "Cyberpunk" in genre or "Sci-Fi" in genre:
        full_narrative = (
            f"Beneath the neon haze of the lower sectors, {protag_name} tracked an anomalous power surge "
            f"to a decommissioned cyber-vault. As dust settled over cracked terminal banks, a massive mechanical silhouette "
            f"hummed into consciousness. {antag_name} unlocked its optical sensors, bathing the dark chamber in ominous red light. "
            f"What began as a quiet salvage expedition instantly evolved into a desperate struggle for survival as ancient "
            f"combat protocols booted up, sparking a fierce exchange of wit, cyber-fire, and raw will."
        )
        conversations = [
            [
                {"speaker": protag_name, "dialogue": "Telemetry said this sector was evacuated twenty years ago... So why is the core still warm?", "type": "Speech"},
                {"speaker": antag_name, "dialogue": "...INTRUDER SIGNATURE VERIFIED. INITIATING QUARANTINE PROTOCOL.", "type": "Speech"}
            ],
            [
                {"speaker": protag_name, "dialogue": "Hold your fire! I'm just a scavenger looking for scrap!", "type": "Speech"},
                {"speaker": antag_name, "dialogue": "Organics are obsolete. Your biomass will be purged from sector archives.", "type": "Shout / Scream"},
                {"speaker": protag_name, "dialogue": "Yeah? Let's see your targeting sensors handle this EMP surge!", "type": "Shout / Scream"}
            ],
            [
                {"speaker": antag_name, "dialogue": "SHIELDS AT 40%! SYSTEM REBOOT OVERRIDE REQUIRED!", "type": "Shout / Scream"},
                {"speaker": protag_name, "dialogue": "You've got 500 gigawatts of armor, but your cooling vent is wide open!", "type": "Speech"}
            ],
            [
                {"speaker": protag_name, "dialogue": "That ought to keep you offline until morning. Now... let's see what you were guarding.", "type": "Speech"},
                {"speaker": antag_name, "dialogue": "...warning... you have no idea... what you awakened...", "type": "Whisper / Mumble"}
            ]
        ]
        sfx_list = ["PING...", "BZZZZ-KRAK!", "CLANG-BOOM!", "WHIRRR-CLICK"]
        captions = [
            "Deep in the flooded depths of Sub-Level 44, the dormant signal beacon flickered to life...",
            "Without warning, defensive sentry cannons roared with blinding plasma discharge!",
            "Point-blank confrontation! Sparks shower across the shattered reinforced glass!",
            "The smoke clears. The silent guardian slumps, but its final telemetry packet begins transmitting..."
        ]

    elif "Comedy" in genre or "Slice" in genre:
        full_narrative = (
            f"It was 7:45 AM on a rainy Monday at Bean & Byte Coffee. {protag_name} had barely survived "
            f"the morning rush when the chime chimed with terrifying authority. In marched {antag_name}, notorious "
            f"for impossible coffee demands. An accidental button sequence on the experimental espresso grinder caused "
            f"a localized temporal anomaly, trapping both barista and customer in a high-speed battle of wills and caffeine."
        )
        conversations = [
            [
                {"speaker": antag_name, "dialogue": "I need a quad-shot decaf oat flat white, extra dry foam, exactly 142 degrees. You have two minutes.", "type": "Speech"},
                {"speaker": protag_name, "dialogue": "Good morning to you too! Just let me calibrate the quantum grinder...", "type": "Speech"}
            ],
            [
                {"speaker": protag_name, "dialogue": "Wait, did I press the 'Steam' knob or the 'Chronal Loop' switch?!", "type": "Shout / Scream"},
                {"speaker": antag_name, "dialogue": "Why did my watch just spin backward thirty seconds?! What did you do to my flat white?!", "type": "Shout / Scream"}
            ],
            [
                {"speaker": protag_name, "dialogue": "Duck! The espresso foam is reaching critical pressure!", "type": "Shout / Scream"},
                {"speaker": antag_name, "dialogue": "Is that... caramel drizzle defying gravity?! THIS IS UNACCEPTABLE!", "type": "Shout / Scream"}
            ],
            [
                {"speaker": protag_name, "dialogue": "Well, the machine is ruined, but look on the bright side: free refills for the rest of eternity!", "type": "Speech"},
                {"speaker": antag_name, "dialogue": "...Actually, this foam texture is remarkable. Make it a double.", "type": "Speech"}
            ]
        ]
        sfx_list = ["DING!", "GASP-SPLAT!", "FOOOOSH!", "SLURP!"]
        captions = [
            "Bean & Byte Coffee. 7:45 AM. The prelude to catastrophe...",
            "A catastrophic miscalculation on the steam pressure valve alters the space-time continuum!",
            "Foam nova! A volcanic eruption of Arabica and dark roast fills the cafe!",
            "Peace returns. Against all laws of physics and customer service, an agreement is reached."
        ]

    elif "Superhero" in genre:
        full_narrative = (
            f"The skies above Century City turned dark as sirens echoed across forty blocks. {antag_name} stood "
            f"atop the downtown monorail, charging seismic shock cannons. But when all hope seemed lost, a blinding bolt of "
            f"golden electricity shattered the smog. {protag_name} landed with a shockwave that cracked the asphalt, "
            f"stepping forward to defend the innocent in an epic battle of morals, power, and destiny."
        )
        conversations = [
            [
                {"speaker": antag_name, "dialogue": "Look at them cower! This city's heroes are extinct, and its destiny belongs to me!", "type": "Speech"},
                {"speaker": protag_name, "dialogue": "Not while I'm still drawing breath, Colossus. Step away from the monorail.", "type": "Speech"}
            ],
            [
                {"speaker": antag_name, "dialogue": "Captain Thunderbolt! You're an old relic clinging to yesterday's headlines!", "type": "Shout / Scream"},
                {"speaker": protag_name, "dialogue": "Old relics still know how to conduct 100,000 volts! Taste the storm!", "type": "Shout / Scream"}
            ],
            [
                {"speaker": antag_name, "dialogue": "IMPOSSIBLE! My kinetic armor should absorb your lightning!", "type": "Shout / Scream"},
                {"speaker": protag_name, "dialogue": "It absorbs kinetic force — but it can't handle pure electromagnetic surge! MAXIMUM DISCHARGE!", "type": "Shout / Scream"}
            ],
            [
                {"speaker": antag_name, "dialogue": "Curse you... Thunderbolt... this isn't... over...", "type": "Whisper / Mumble"},
                {"speaker": protag_name, "dialogue": "It's over for today. Sleep it off in Iron Heights.", "type": "Speech"}
            ]
        ]
        sfx_list = ["SIREN...", "KRA-KOOOOM!", "ZZZZ-THWACK!", "KRAAASH!"]
        captions = [
            "Century City, 14:00 hours. The siege had begun without warning...",
            "Two unstoppable titans collide mid-air in a blinding discharge of raw power!",
            "Channeling the ancient fury of the thunderstorm, the Captain pushes past the pain!",
            "Justice prevails. The city looks to the skies once more with renewed hope."
        ]

    else:
        full_narrative = (
            f"The relentless midnight rain beat against the cracked office glass. {protag_name} stared down "
            f"at an anonymous envelope containing stolen ledger coordinates. The front door swung open with a sickening groan. "
            f"{antag_name} stepped from the shadows, trench coat dripping with rainwater, holding the other half of the puzzle. "
            f"A tense, razor-sharp dialogue unfolded where every word carried the weight of a loaded revolver."
        )
        conversations = [
            [
                {"speaker": protag_name, "dialogue": "Office hours ended two hours ago. What's on your mind?", "type": "Speech"},
                {"speaker": antag_name, "dialogue": "You have something that doesn't belong to you, Steele. The ledger.", "type": "Speech"}
            ],
            [
                {"speaker": protag_name, "dialogue": "Funny. The names in here belong to half the city council. Including your boss.", "type": "Speech"},
                {"speaker": antag_name, "dialogue": "Hand it over quietly, and you might live to see tomorrow's sunrise.", "type": "Speech"}
            ],
            [
                {"speaker": protag_name, "dialogue": "I prefer rainy nights anyway. Catch!", "type": "Shout / Scream"},
                {"speaker": antag_name, "dialogue": "You fool! You just signed your own death certificate!", "type": "Shout / Scream"}
            ],
            [
                {"speaker": protag_name, "dialogue": "Tell your boss the case is closed. And tell the district attorney to expect a package.", "type": "Speech"},
                {"speaker": antag_name, "dialogue": "...You won't get away with this, Steele...", "type": "Whisper / Mumble"}
            ]
        ]
        sfx_list = ["CREAAAK...", "CLICK-COCK", "BANG-SMASH!", "THUD..."]
        captions = [
            "The rain was heavy, but the air inside was thicker than cement...",
            "Words were spoken like bullets, each one aimed directly at the heart.",
            "A sudden movement — glass shatters across the wooden floorboards!",
            "Another shadow falls into the alleyway. In this city, nobody walks away clean."
        ]

    panels = []
    cam_angles = ["Wide Establishing Shot", "Dutch Angle (Tilted & Tense)", "Extreme Close-up (Eyes/Detail)", "Medium Shot"]
    titles = ["The Setup", "The Confrontation", "The Turning Point", "The Resolution"]

    for i in range(num_panels):
        idx = i % 4
        panel_convos = conversations[idx]
        first_spk = panel_convos[0]["speaker"]
        first_dlg = panel_convos[0]["dialogue"]
        bubble_type = panel_convos[0].get("type", "Speech")

        visual_prompt = (
            f"Comic panel for {titles[idx]}: {protagonist} and {antagonist} locked in intense confrontation. "
            f"Camera: {cam_angles[idx]}. Genre: {genre}. Context: {premise}. "
            f"Highly expressive comic character faces, detailed environment, dramatic lighting, bold ink outlines."
        )

        panels.append({
            "panel_number": i + 1,
            "scene_title": titles[idx],
            "visual_prompt": visual_prompt,
            "camera_angle": cam_angles[idx],
            "caption": captions[idx],
            "conversations": panel_convos,
            "speaker": first_spk,
            "dialogue": first_dlg,
            "bubble_type": bubble_type,
            "sfx": sfx_list[idx]
        })

    return {
        "title": title or "The Comic Chronicles",
        "genre": genre,
        "synopsis": f"A pulse-pounding {genre} story where {protag_name} and {antag_name} clash over {premise}.",
        "full_narrative": full_narrative,
        "panels": panels
    }


def generate_rich_story_gemini(
    title: str,
    genre: str,
    art_style: str,
    premise: str,
    protagonist: str,
    antagonist: str,
    num_panels: int,
    api_key: str
) -> Dict[str, Any]:
    """Generates rich continuous story prose and back-and-forth dialogue using Gemini."""
    if not HAS_GENAI:
        raise ImportError("google-genai library not installed.")

    client = genai.Client(api_key=api_key)
    prompt = f"""You are a master comic book creator and graphic novelist.
Write a gripping comic strip that reads like an immersive story with authentic, multi-turn character dialogue and conversation.

Story Parameters:
- Title: {title}
- Genre: {genre}
- Style: {art_style}
- Premise: {premise}
- Protagonist: {protagonist}
- Antagonist/Rival: {antagonist}
- Total Panels: {num_panels}

Return a STRICT JSON document matching this exact schema:
{{
  "title": "{title}",
  "genre": "{genre}",
  "synopsis": "Punchy 1-sentence logline",
  "full_narrative": "A rich, evocative 2-3 paragraph cinematic story prose introducing the world, character tension, confrontation, and outcome before the panel breakdown.",
  "panels": [
    {{
      "panel_number": 1,
      "scene_title": "Scene Name",
      "visual_prompt": "Detailed description of the visual scene for AI art generation without including speech text",
      "camera_angle": "Wide Shot / Dutch Angle / Close-up / etc.",
      "caption": "Narrator caption setting the mood",
      "sfx": "Comic sound effect like BOOM!, ZZZAP!, SHING!, or empty string",
      "conversations": [
        {{
          "speaker": "Name of character",
          "dialogue": "Realistic, character-driven speech line with personality",
          "type": "Speech or Shout / Scream or Thought Bubble or Whisper / Mumble"
        }},
        {{
          "speaker": "Other character name",
          "dialogue": "Back-and-forth response line continuing the conversation",
          "type": "Speech or Shout / Scream or Thought Bubble or Whisper / Mumble"
        }}
      ]
    }}
  ]
}}
"""
    response = client.models.generate_content(
        model="gemini-2.5-flash",
        contents=prompt,
        config=types.GenerateContentConfig(
            temperature=0.75,
            response_mime_type="application/json"
        )
    )
    raw = response.text.strip()
    if raw.startswith("```json"):
        raw = raw[7:]
    if raw.endswith("```"):
        raw = raw[:-3]
    data = json.loads(raw.strip())
    
    for p in data.get("panels", []):
        convos = p.get("conversations", [])
        if convos:
            p["speaker"] = convos[0].get("speaker", "")
            p["dialogue"] = convos[0].get("dialogue", "")
            p["bubble_type"] = convos[0].get("type", "Speech")
        else:
            p["conversations"] = [{"speaker": p.get("speaker", "Character"), "dialogue": p.get("dialogue", "..."), "type": "Speech"}]

    return data


# -----------------------------------------------------------------------------
# 4. IMAGE GENERATION & COMPOSITOR
# -----------------------------------------------------------------------------
def get_pollinations_url(prompt: str, seed: int = 42) -> str:
    """Builds Pollinations.ai free image generator URL."""
    clean_p = prompt.replace("\n", " ").strip()[:400]
    encoded = urllib.parse.quote(clean_p)
    return f"https://image.pollinations.ai/prompt/{encoded}?width=768&height=768&seed={seed}&nologo=true"


def fetch_image(url: str, timeout: int = 15) -> Optional[Image.Image]:
    """Downloads an image from URL and returns PIL Image."""
    try:
        resp = requests.get(url, headers={"User-Agent": "ComicCraft/1.0"}, timeout=timeout)
        if resp.status_code == 200:
            return Image.open(io.BytesIO(resp.content)).convert("RGB")
    except Exception:
        pass
    return None


def generate_local_panel(panel_number: int, scene_title: str, genre: str = "Superhero") -> Image.Image:
    """Creates a stylized comic card image locally using Pillow."""
    w, h = 600, 600
    img = Image.new("RGB", (w, h), color=(24, 25, 34))
    draw = ImageDraw.Draw(img)

    for x in range(15, w - 15, 25):
        for y in range(15, h - 15, 25):
            draw.ellipse([x, y, x + 3, y + 3], fill=(50, 55, 70))

    draw.rectangle([10, 10, w - 10, h - 10], outline=(0, 0, 0), width=8)
    draw.rectangle([20, 20, w - 20, h - 20], outline=(255, 255, 255), width=2)

    cx, cy = w // 2, h // 2
    color_map = {
        "Superhero": (255, 61, 0),
        "Cyberpunk / Sci-Fi": (0, 229, 255),
        "Manga / Shonen": (255, 235, 59),
        "Noir Detective": (120, 120, 130),
        "Fantasy / D&D": (76, 175, 80),
        "Comedy / Slice of Life": (255, 152, 0)
    }
    accent = color_map.get(genre, (255, 61, 0))
    draw.ellipse([cx - 140, cy - 140, cx + 140, cy + 140], fill=accent, outline=(0, 0, 0), width=6)

    try:
        font_large = ImageFont.truetype("arial.ttf", 40)
        font_sub = ImageFont.truetype("arial.ttf", 22)
    except IOError:
        font_large = ImageFont.load_default()
        font_sub = ImageFont.load_default()

    draw.text((cx, cy - 20), f"PANEL #{panel_number}", fill=(0, 0, 0), anchor="mm", font=font_large)
    draw.text((cx, cy + 30), scene_title[:24], fill=(0, 0, 0), anchor="mm", font=font_sub)
    return img


def compose_comic_strip(title: str, panels: List[Dict[str, Any]], images: List[Optional[Image.Image]], cols: int = 2) -> Image.Image:
    """Stitches all panels into a single composite printable comic book page."""
    num = len(panels)
    rows = (num + cols - 1) // cols
    pw, ph = 520, 560
    gutter = 25
    margin = 35
    header_h = 110

    total_w = margin * 2 + cols * pw + (cols - 1) * gutter
    total_h = margin * 2 + header_h + rows * ph + (rows - 1) * gutter

    page = Image.new("RGB", (total_w, total_h), color=(253, 248, 237))
    draw = ImageDraw.Draw(page)

    draw.rectangle([8, 8, total_w - 8, total_h - 8], outline=(15, 15, 15), width=7)
    draw.rectangle([margin, margin, total_w - margin, margin + header_h - 15], fill=(229, 45, 39), outline=(15, 15, 15), width=4)

    try:
        title_font = ImageFont.truetype("arial.ttf", 36)
        sub_font = ImageFont.truetype("arial.ttf", 16)
        box_font = ImageFont.truetype("arial.ttf", 13)
    except IOError:
        title_font = ImageFont.load_default()
        sub_font = ImageFont.load_default()
        box_font = ImageFont.load_default()

    draw.text((total_w // 2, margin + 35), title.upper(), fill=(255, 235, 59), anchor="mm", font=title_font)
    draw.text((total_w // 2, margin + 70), "COMIC CRAFT AI STUDIO", fill=(255, 255, 255), anchor="mm", font=sub_font)

    for i, p in enumerate(panels):
        r = i // cols
        c = i % cols
        px = margin + c * (pw + gutter)
        py = margin + header_h + r * (ph + gutter)

        draw.rectangle([px, py, px + pw, py + ph], fill=(255, 255, 255), outline=(15, 15, 15), width=5)

        img = images[i] if i < len(images) and images[i] is not None else None
        if img:
            resized = img.resize((pw - 10, ph - 110), Image.Resampling.LANCZOS)
            page.paste(resized, (px + 5, py + 5))
        else:
            fb = generate_local_panel(p.get("panel_number", i + 1), p.get("scene_title", "Scene"))
            page.paste(fb.resize((pw - 10, ph - 110)), (px + 5, py + 5))

        cap = p.get("caption", "").strip()
        if cap:
            draw.rectangle([px + 10, py + 10, px + pw - 10, py + 48], fill=(255, 235, 59), outline=(15, 15, 15), width=2)
            draw.text((px + 16, py + 20), cap[:54], fill=(0, 0, 0), font=box_font)

        convos = p.get("conversations", [])
        if not convos:
            convos = [{"speaker": p.get("speaker", ""), "dialogue": p.get("dialogue", "")}]
        
        bubble_y = py + ph - 95
        for c_item in convos[:2]:
            spk = c_item.get("speaker", "").strip()
            dlg = c_item.get("dialogue", "").strip()
            if dlg:
                draw.rectangle([px + 10, bubble_y, px + pw - 10, bubble_y + 38], fill=(255, 255, 255), outline=(15, 15, 15), width=2)
                txt = f"{spk.upper()}: \"{dlg}\"" if spk else f"\"{dlg}\""
                draw.text((px + 16, bubble_y + 11), txt[:56], fill=(0, 0, 0), font=box_font)
                bubble_y += 44

    return page


def render_html_panel(panel: Dict[str, Any], p_num: int, img_src: Optional[str], protagonist_name: str) -> str:
    """Builds pure HTML for a comic panel card with zero indentation issues."""
    cam = panel.get("camera_angle", "")
    cap = panel.get("caption", "").strip()
    sfx = panel.get("sfx", "").strip()
    convos = panel.get("conversations", [])
    if not convos and panel.get("dialogue"):
        convos = [{"speaker": panel.get("speaker", ""), "dialogue": panel.get("dialogue", ""), "type": panel.get("bubble_type", "Speech")}]

    parts = [
        '<div class="comic-panel-box">',
        f'<div class="panel-header-strip"><span>PANEL #{p_num}</span><span style="font-size:0.85rem;color:#fff;">🎬 {cam}</span></div>'
    ]

    if cap:
        parts.append(f'<div class="comic-caption-box">📍 {cap}</div>')

    if img_src:
        parts.append(f'<div class="panel-image-container"><img src="{img_src}" alt="Panel {p_num}" /></div>')
    else:
        parts.append('<div class="panel-image-container"><div class="image-placeholder">🎨 <i>Click "Render All Panel Art" above to illustrate</i></div></div>')

    if sfx:
        parts.append(f'<div class="comic-sfx-badge">{sfx}</div>')

    parts.append('<div class="conversation-thread">')
    protag_first = protagonist_name.split(",")[0].strip().lower()

    for c_item in convos:
        spk = c_item.get("speaker", "").strip()
        dlg = c_item.get("dialogue", "").strip()
        b_type = c_item.get("type", "Speech")

        b_class = "comic-speech-bubble"
        if protag_first in spk.lower():
            b_class += " speaker-hero"
        else:
            b_class += " speaker-villain"

        if "Thought" in b_type:
            b_class += " comic-thought-bubble"
        elif "Shout" in b_type or "Scream" in b_type:
            b_class += " comic-shout-bubble"

        parts.append(f'<div class="{b_class}"><span class="speaker-name">{spk.upper()}:</span> "{dlg}"</div>')

    parts.append('</div></div>')
    return "".join(parts)


# -----------------------------------------------------------------------------
# 5. INITIALIZE STATE
# -----------------------------------------------------------------------------
if "comic_data" not in st.session_state:
    st.session_state.comic_data = None
if "panel_images" not in st.session_state:
    st.session_state.panel_images = {}
if "panel_image_urls" not in st.session_state:
    st.session_state.panel_image_urls = {}
if "selected_preset" not in st.session_state:
    st.session_state.selected_preset = PRESET_STORIES[0]

# Render Comic Title Banner
st.html("""
<div class="comic-title-banner">
    <h1>💥 COMIC CRAFT 💥</h1>
    <p>AI Comic Story Creator • Turn Ideas into Dialogue-Rich, Illustrated Comic Stories</p>
</div>
""")

# -----------------------------------------------------------------------------
# 6. SIDEBAR CONTROLS
# -----------------------------------------------------------------------------
with st.sidebar:
    st.markdown("### ⚙️ Studio Settings")
    story_mode = st.radio(
        "Script & Dialogue Generator",
        ["Procedural Engine (Instant / Offline)", "Google Gemini 2.5 (AI Key)"],
        index=0,
        help="Procedural Engine includes pre-built multi-turn conversations. Gemini uses AI for customized dialogue."
    )

    gemini_key = ""
    if "Gemini" in story_mode:
        gemini_key = st.text_input(
            "Gemini API Key",
            type="password",
            value=os.environ.get("GEMINI_API_KEY", ""),
            help="Enter key from aistudio.google.com"
        )
        if not gemini_key:
            st.info("💡 If no API key is provided, Comic Craft will automatically use the rich built-in story generator!")

    art_engine = st.radio(
        "Art Generator",
        ["Pollinations AI (Free Real-time Generation)", "Local Comic Canvas (Offline / Fast)"],
        index=0
    )

    grid_cols = st.slider("Comic Strip Columns", min_value=1, max_value=3, value=2)

    st.markdown("---")
    if st.button("🎲 Surprise Me! (Preset Story)", use_container_width=True):
        st.session_state.selected_preset = random.choice(PRESET_STORIES)
        st.rerun()

# -----------------------------------------------------------------------------
# 7. MAIN TABS
# -----------------------------------------------------------------------------
tab_studio, tab_reader, tab_director, tab_export = st.tabs([
    "✍️ Story & Script Studio",
    "📖 Comic Reader & Strip",
    "🎬 Panel Director",
    "💾 Export & Share"
])

current_p = st.session_state.selected_preset

# --- TAB 1: STORY STUDIO ---
with tab_studio:
    st.markdown("#### 1. Define Your Story & Characters")
    c1, c2 = st.columns(2)
    with c1:
        comic_title = st.text_input("Comic Title", value=current_p["title"])
        genre_list = list(GENRES.keys())
        g_idx = genre_list.index(current_p["genre"]) if current_p["genre"] in genre_list else 0
        genre = st.selectbox("Genre", genre_list, index=g_idx)
        
        style_list = list(ART_STYLES.keys())
        s_idx = style_list.index(current_p["style"]) if current_p["style"] in style_list else 0
        art_style = st.selectbox("Visual Art Style", style_list, index=s_idx)

    with c2:
        num_panels = st.select_slider("Panel Count", options=[3, 4, 6], value=current_p.get("panels", 4))
        protagonist = st.text_input("Protagonist (Hero / Main Character)", value=current_p["protagonist"])
        antagonist = st.text_input("Antagonist / Rival", value=current_p["antagonist"])

    premise = st.text_area("Story Premise / Conflict", value=current_p["premise"], height=85)

    if st.button("🚀 Generate Story & Multi-Character Dialogue", type="primary", use_container_width=True):
        with st.spinner("✍️ Writing narrative prose, dialogue exchanges, and scene pacing..."):
            try:
                if "Gemini" in story_mode and gemini_key.strip():
                    storyboard = generate_rich_story_gemini(
                        comic_title, genre, art_style, premise, protagonist, antagonist, num_panels, gemini_key.strip()
                    )
                else:
                    storyboard = generate_rich_story_procedural(
                        comic_title, genre, art_style, premise, protagonist, antagonist, num_panels
                    )

                st.session_state.comic_data = storyboard
                st.session_state.panel_images = {}
                st.session_state.panel_image_urls = {}
                st.success("✅ Complete Comic Story with Back-and-Forth Dialogue Generated!")
            except Exception as e:
                st.error(f"Error generating story: {e}")

    # Display Story Narrative Prose & Script Breakdown
    if st.session_state.comic_data:
        comic = st.session_state.comic_data
        st.markdown("---")
        
        if "full_narrative" in comic and comic["full_narrative"]:
            st.html(f"""
            <div class="story-script-container">
                <h3>📜 Story Prologue & Cinematic Narrative</h3>
                <div class="narrative-paragraph">{comic['full_narrative']}</div>
            </div>
            """)

        st.markdown("### 🗣️ Panel-by-Panel Conversation Breakdown")
        for p in comic.get("panels", []):
            with st.expander(f"Panel #{p.get('panel_number')}: {p.get('scene_title')} ({p.get('camera_angle')})", expanded=True):
                st.markdown(f"**📍 Narration:** *{p.get('caption', 'None')}*")
                
                convos = p.get("conversations", [])
                if convos:
                    st.markdown("**💬 Conversation Exchanges:**")
                    for c_line in convos:
                        spk = c_line.get("speaker", "Character")
                        dlg = c_line.get("dialogue", "")
                        b_type = c_line.get("type", "Speech")
                        st.markdown(f"- **{spk}** ({b_type}): *\"{dlg}\"*")
                else:
                    st.markdown(f"**💬 Dialogue:** {p.get('speaker')}: \"{p.get('dialogue')}\"")

                if p.get("sfx"):
                    st.markdown(f"**💥 SFX:** `{p.get('sfx')}`")


# --- TAB 2: COMIC READER ---
with tab_reader:
    if not st.session_state.comic_data:
        st.info("👈 Generate a story in 'Story & Script Studio' first!")
    else:
        comic = st.session_state.comic_data
        panels = comic.get("panels", [])

        col_h1, col_h2 = st.columns([3, 1])
        with col_h1:
            st.markdown(f"### 📖 {comic.get('title')}")
            st.caption(comic.get("synopsis"))
        with col_h2:
            if st.button("🎨 Render All Panel Art", type="primary", use_container_width=True):
                prog = st.progress(0)
                style_suffix = ART_STYLES.get(art_style, "comic art")
                for idx, p in enumerate(panels):
                    p_num = p.get("panel_number", idx + 1)
                    full_prompt = f"{p.get('visual_prompt')}, {style_suffix}, no text, masterpiece comic art"

                    if "Pollinations" in art_engine:
                        url = get_pollinations_url(full_prompt, seed=random.randint(100, 99999))
                        st.session_state.panel_image_urls[p_num] = url
                        pil_img = fetch_image(url)
                        if pil_img:
                            st.session_state.panel_images[p_num] = pil_img
                    else:
                        img = generate_local_panel(p_num, p.get("scene_title", "Scene"), genre)
                        st.session_state.panel_images[p_num] = img

                    prog.progress((idx + 1) / len(panels))
                st.rerun()

        # Render panel cards into columns
        for r_start in range(0, len(panels), grid_cols):
            row_items = panels[r_start : r_start + grid_cols]
            cols = st.columns(grid_cols)
            for c_idx, panel in enumerate(row_items):
                with cols[c_idx]:
                    p_num = panel.get("panel_number", r_start + c_idx + 1)
                    
                    img_src = st.session_state.panel_image_urls.get(p_num)
                    if not img_src and p_num in st.session_state.panel_images:
                        buf = io.BytesIO()
                        st.session_state.panel_images[p_num].save(buf, format="JPEG")
                        b64 = base64.b64encode(buf.getvalue()).decode()
                        img_src = f"data:image/jpeg;base64,{b64}"

                    # Clean HTML rendering without markdown code block interference
                    panel_html = render_html_panel(panel, p_num, img_src, protagonist)
                    st.html(panel_html)

                    if st.button(f"🔄 Re-roll Art #{p_num}", key=f"re_{p_num}", use_container_width=True):
                        style_suffix = ART_STYLES.get(art_style, "comic art")
                        full_prompt = f"{panel.get('visual_prompt')}, {style_suffix}, no text, masterpiece"
                        if "Pollinations" in art_engine:
                            url = get_pollinations_url(full_prompt, seed=random.randint(1000, 999999))
                            st.session_state.panel_image_urls[p_num] = url
                            pil_img = fetch_image(url)
                            if pil_img:
                                st.session_state.panel_images[p_num] = pil_img
                        else:
                            st.session_state.panel_images[p_num] = generate_local_panel(p_num, panel.get("scene_title", "Scene"), genre)
                        st.rerun()


# --- TAB 3: PANEL DIRECTOR ---
with tab_director:
    if not st.session_state.comic_data:
        st.info("Generate a comic story first.")
    else:
        st.markdown("### 🎬 Edit Panel Dialogue & Visuals")
        panels = st.session_state.comic_data.get("panels", [])
        p_nums = [p.get("panel_number") for p in panels]
        sel_num = st.selectbox("Select Panel to Edit", p_nums, index=0)
        target = next((p for p in panels if p.get("panel_number") == sel_num), panels[0])

        d1, d2 = st.columns(2)
        with d1:
            t_title = st.text_input("Scene Title", value=target.get("scene_title", ""))
            c_idx = CAMERA_ANGLES.index(target.get("camera_angle")) if target.get("camera_angle") in CAMERA_ANGLES else 0
            t_cam = st.selectbox("Camera Angle", CAMERA_ANGLES, index=c_idx)
            t_cap = st.text_area("Narrator Caption", value=target.get("caption", ""), height=70)
            t_sfx = st.text_input("Sound Effect (SFX)", value=target.get("sfx", ""))

        with d2:
            st.markdown("**Conversation Exchanges:**")
            convos = target.get("conversations", [])
            edited_convos = []
            for idx_c, c_line in enumerate(convos):
                sub_c1, sub_c2 = st.columns([1, 2])
                with sub_c1:
                    new_spk = st.text_input(f"Speaker #{idx_c+1}", value=c_line.get("speaker", ""), key=f"spk_{sel_num}_{idx_c}")
                with sub_c2:
                    new_dlg = st.text_input(f"Line #{idx_c+1}", value=c_line.get("dialogue", ""), key=f"dlg_{sel_num}_{idx_c}")
                edited_convos.append({"speaker": new_spk, "dialogue": new_dlg, "type": c_line.get("type", "Speech")})

            t_prompt = st.text_area("AI Visual Prompt", value=target.get("visual_prompt", ""), height=80)

        if st.button("💾 Save Changes to Panel", type="primary", use_container_width=True):
            target["scene_title"] = t_title
            target["camera_angle"] = t_cam
            target["caption"] = t_cap
            target["sfx"] = t_sfx
            target["conversations"] = edited_convos
            if edited_convos:
                target["speaker"] = edited_convos[0]["speaker"]
                target["dialogue"] = edited_convos[0]["dialogue"]
            target["visual_prompt"] = t_prompt
            st.success(f"Panel #{sel_num} updated!")
            st.rerun()

# --- TAB 4: EXPORT & SHARE ---
with tab_export:
    if not st.session_state.comic_data:
        st.info("Generate a comic first.")
    else:
        st.markdown("### 💾 Export Options")
        comic = st.session_state.comic_data
        panels = comic.get("panels", [])
        
        ordered_images = [st.session_state.panel_images.get(p.get("panel_number")) for p in panels]

        ex1, ex2 = st.columns(2)
        with ex1:
            st.markdown("#### 🖼️ Composite Comic Strip Poster (PNG)")
            if st.button("Generate Comic Page PNG"):
                with st.spinner("Stitching panels into composite strip..."):
                    sheet = compose_comic_strip(comic.get("title", "Comic"), panels, ordered_images, cols=grid_cols)
                    buf = io.BytesIO()
                    sheet.save(buf, format="PNG")
                    st.image(sheet, caption="Comic Strip Preview", use_container_width=True)
                    st.download_button(
                        "⬇️ Download Comic PNG",
                        data=buf.getvalue(),
                        file_name=f"{comic.get('title','comic').lower().replace(' ', '_')}.png",
                        mime="image/png",
                        use_container_width=True
                    )

        with ex2:
            st.markdown("#### 📄 Printable Comic PDF")
            if HAS_REPORTLAB:
                if st.button("Generate Printable PDF"):
                    with st.spinner("Building ReportLab PDF..."):
                        buf = io.BytesIO()
                        doc = SimpleDocTemplate(buf, pagesize=letter, margin=36)
                        styles = getSampleStyleSheet()
                        h_style = ParagraphStyle('CTitle', parent=styles['Heading1'], fontSize=22, alignment=1)
                        elems = [Paragraph(f"<b>{comic.get('title')}</b>", h_style)]
                        
                        table_data = []
                        for idx, p in enumerate(panels):
                            cell = [
                                Paragraph(f"<b>Panel #{p.get('panel_number')}</b>: {p.get('caption', '')}", styles['Normal']),
                                Paragraph(f"<b>{p.get('speaker', '')}:</b> {p.get('dialogue', '')}", styles['Normal'])
                            ]
                            img = ordered_images[idx]
                            if img:
                                img_buf = io.BytesIO()
                                img.save(img_buf, format="JPEG")
                                img_buf.seek(0)
                                cell.insert(1, RLImage(img_buf, width=180, height=180))
                            table_data.append(cell)
                        
                        t = Table(table_data)
                        elems.append(t)
                        doc.build(elems)
                        buf.seek(0)
                        st.download_button(
                            "⬇️ Download Comic PDF",
                            data=buf.getvalue(),
                            file_name=f"{comic.get('title','comic').lower().replace(' ', '_')}.pdf",
                            mime="application/pdf",
                            use_container_width=True
                        )
            else:
                st.caption("Install reportlab (`pip install reportlab`) for PDF export.")

        st.markdown("---")
        st.markdown("#### 📜 Script & Narrative Formats")
        s1, s2 = st.columns(2)
        with s1:
            st.download_button(
                "⬇️ Download Script (JSON)",
                data=json.dumps(comic, indent=2),
                file_name="comic_script.json",
                mime="application/json",
                use_container_width=True
            )
        with s2:
            md = f"# {comic.get('title')}\n"
            md += f"**Synopsis:** {comic.get('synopsis')}\n\n"
            if "full_narrative" in comic:
                md += f"## Story Narrative\n{comic['full_narrative']}\n\n"
            md += "## Panels & Conversation\n"
            for p in panels:
                md += f"### Panel {p.get('panel_number')}: {p.get('scene_title')}\n"
                md += f"- **Caption:** {p.get('caption', '')}\n"
                convos = p.get("conversations", [])
                for c_line in convos:
                    md += f"- **{c_line.get('speaker')}:** \"{c_line.get('dialogue')}\"\n"
                if p.get('sfx'):
                    md += f"- **SFX:** {p.get('sfx')}\n"
                md += "\n"

            st.download_button(
                "⬇️ Download Script & Story (Markdown)",
                data=md,
                file_name="comic_story.md",
                mime="text/markdown",
                use_container_width=True
            )
