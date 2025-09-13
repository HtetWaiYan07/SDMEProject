import requests

# Reference purine levels (mg/100g) by food category
purine_reference = {
    'organ meats': 400,
    'red meat': 150,
    'poultry': 120,
    'seafood': 200,
    'legumes': 70,
    'tofu': 30,
    'dairy': 10,
    'fruits': 5,
    'vegetables': 20,
    'grains': 10,
    'sauces/condiments': 10,  # low-purine default for plant-based sauces
    'unknown': 10
}

# Legacy exact tags
category_map = {
    "meats": "red meat", "beef": "red meat", "pork": "red meat", "veal": "red meat",
    "poultry": "poultry", "chicken": "poultry", "turkey": "poultry",
    "fish": "seafood", "fishes": "seafood", "crustaceans": "seafood", "seafood": "seafood",
    "tofu": "tofu", "soy": "legumes", "soybeans": "legumes",
    "cheese": "dairy", "milk": "dairy", "yogurt": "dairy",
    "fruits": "fruits", "vegetables": "vegetables", "legumes": "legumes",
    "cereals": "grains", "bread": "grains", "rice": "grains",
    "sauces": "sauces/condiments", "pesto": "sauces/condiments"
}

# Expanded triggers (namespaced tags)
CATEGORY_TRIGGERS = [
    ("organ meats", {"organ-meats","offal","liver","livers","liver-pate","liver-pâté","pate","pâté",
                     "kidney","kidneys","heart","sweetbreads","giblets","tongue"}),
    ("seafood", {"seafood","fish","fishes","crustaceans","shellfish","sardines","sardine",
                 "anchovies","anchovy","mackerel","herring","trout","tuna","mussels",
                 "scallops","shrimp","prawns","lobster","octopus","squid","clams","clam"}),
    ("red meat", {"meats","beef","pork","veal","mutton","lamb"}),
    ("poultry", {"poultry","chicken","turkey","duck","goose"}),
    ("legumes", {"legumes","soy","soybeans","lentils","peas","beans"}),
    ("tofu", {"tofu"}),
    ("dairy", {"cheese","milk","yogurt"}),
    ("fruits", {"fruits"}),
    ("vegetables", {"vegetables","asparagus","spinach","cauliflower","mushroom","mushrooms","peas"}),
    ("grains", {"cereals","bread","rice","grains"}),
    ("sauces/condiments", {"sauces","condiments","pesto","pesto-genovese","pesto-alla-genovese","salsa","paste","spread"})
]

# Moderate‑purine vegetables (flag and modest effect)
MODERATE_VEG = {"mushroom","mushrooms","spinach","asparagus","peas","cauliflower"}

# Processed context tags (for extra caution)
PROCESSED_TAGS = {"sauces","condiments","pesto","pesto-genovese","pesto-alla-genovese","salsa","paste","spread","ready-meals","prepared","processed"}

# Tomato tokens (possible trigger for some)
TOMATO_TOKENS = {"tomato","tomatoes"}

def _normalize_tag(tag):
    t = (tag or "").strip().lower()
    if ":" in t:
        lang, suffix = t.split(":", 1)
        return t, suffix
    return t, t

def _infer_processed(categories, ingredients_text):
    # Detect processed context from tags or keywords
    lower_ing = (ingredients_text or "").lower()
    for raw in (categories or []):
        full, suffix = _normalize_tag(raw)
        if full in PROCESSED_TAGS or suffix in PROCESSED_TAGS:
            return True
    for kw in ["paste","spread","sauce","condiment","emulsifier","stabilizer","ready meal","instant","powder"]:
        if kw in lower_ing:
            return True
    return False

def _map_category(categories, ingredients_text):
    hits = set()
    for raw in (categories or []):
        full, suffix = _normalize_tag(raw)
        if full in category_map: hits.add(category_map[full])
        if suffix in category_map: hits.add(category_map[suffix])
        for group, triggers in CATEGORY_TRIGGERS:
            if full in triggers or suffix in triggers:
                hits.add(group)

    # Ingredient fallback for organ meats
    ing = (ingredients_text or "").lower()
    organ_cues = ["liver","pâté","pate","offal","giblets","sweetbread","kidney","heart","tongue"]
    if any(c in ing for c in organ_cues):
        hits.add("organ meats")

    # If nothing matched, infer plant sauce if pesto-like, else sauces/condiments as safe default
    if not hits:
        if any(k in ing for k in ["basil","olive oil","pine nuts","parmesan","pecorino","garlic"]):
            hits.add("sauces/condiments")
        else:
            hits.add("sauces/condiments")  # avoid 'unknown' classification

    # Choose highest purine baseline to avoid underestimation
    return max(hits, key=lambda g: purine_reference.get(g, 0))

# --- Fetch product info ---
def fetch_off_product(barcode):
    # OFF API product read; fields per OFF docs
    url = f"https://world.openfoodfacts.org/api/v2/product/{barcode}.json"
    try:
        response = requests.get(url, timeout=15)
    except requests.RequestException:
        return None
    if response.status_code == 200:
        return response.json()
    return None

# --- Adjusted protein + nucleotides ---
def get_adjusted_protein_and_nucleotides(nutriments):
    proteins = nutriments.get("proteins_100g", 0) or 0
    collagen_ratio = nutriments.get("collagen-meat-protein-ratio_100g", 0) or 0
    serum_protein = nutriments.get("serum-proteins_100g", 0) or 0
    whey = nutriments.get("whey-proteins_100g", 0) or 0
    casein = nutriments.get("casein_100g", 0) or 0
    nucleotides = nutriments.get("nucleotides_100g", 0) or 0
    adjusted = (
        proteins * 1.0 +
        collagen_ratio * 0.5 +
        serum_protein * 1.2 +
        whey * 1.1 +
        casein * 1.0
    )
    return round(adjusted, 2), round(nucleotides, 2)

# --- Purine estimator ---
def estimate_purines(food_category, protein_g, nucleotides_g, nutriments, tokens, is_processed):
    base = purine_reference.get(food_category, purine_reference['sauces/condiments'])

    # Protein (bounded)
    if protein_g > 20:
        base += 15
    elif protein_g < 5:
        base -= 5

    # Direct purines
    adenine = nutriments.get("adenine_100g", 0) or 0
    guanine = nutriments.get("guanine_100g", 0) or 0
    direct_purines = max(0, min(adenine, 800)) + max(0, min(guanine, 800))
    base += direct_purines

    # Nucleotides fallback
    if direct_purines == 0 and nucleotides_g:
        base += min(nucleotides_g, 5.0) * 12

    # Sugars & carbs
    carbs_g = nutriments.get("carbohydrates_100g", 0) or 0
    sugars_g = nutriments.get("sugars_100g", nutriments.get("sugar_100g", 0)) or 0
    fructose_g = nutriments.get("fructose_100g", 0) or 0
    non_fructose = max(0.0, sugars_g - fructose_g)
    base += max(0, min(fructose_g, 25)) * 2.0
    base += max(0, min(non_fructose, 40)) * 0.3
    starchy_g = max(0.0, carbs_g - sugars_g)
    base += max(0, min(starchy_g, 60)) * 0.12

    # Alcohol
    alcohol_g = nutriments.get("alcohol_100g", 0) or 0
    base += max(0, min(alcohol_g, 10)) * 8

    # Vitamin C (protective)
    vitamin_c_mg = nutriments.get("vitamin-c_100g", 0) or 0
    base -= min(vitamin_c_mg * 0.4, 30)

    # Fats
    fat_g = nutriments.get("fat_100g", 0) or 0
    satfat_g = nutriments.get("saturated-fat_100g", nutriments.get("saturatedfat_100g", 0)) or 0
    base += max(0, min(fat_g, 50)) * 0.1
    base += max(0, min(satfat_g, 30)) * 0.2

    # Moderate‑purine veg
    if tokens & MODERATE_VEG:
        base += 15 if is_processed else 10

    # Tomato small nudge
    if tokens & TOMATO_TOKENS:
        base += 5 if is_processed else 3

    # Keep high categories high
    if food_category == "organ meats":
        base = max(base, purine_reference['organ meats'] - 20)
    elif food_category == "seafood":
        base = max(base, purine_reference['seafood'] - 20)

    return max(0, round(base))

# --- Risk classification ---
def classify_purine_risk(purine_mg_per_100g):
    if purine_mg_per_100g < 100:
        return "🟢 Low"
    elif 100 <= purine_mg_per_100g <= 200:
        return "🟡 Moderate"
    else:
        return "🔴 High"

# --- Inflammation classifier ---
def classify_inflammation(food_category, nutriments, tokens, is_processed):
    score = 0
    drivers = []

    # Category tilt
    if food_category in ("red meat", "organ meats"):
        score += 2; drivers.append("animal proteins")
    elif food_category in ("seafood", "fruits", "vegetables", "legumes", "tofu", "sauces/condiments"):
        score -= 1; drivers.append("plant-based profile")

    # Macros
    carbs_g = nutriments.get("carbohydrates_100g", 0) or 0
    sugars_g = nutriments.get("sugars_100g", nutriments.get("sugar_100g", 0)) or 0
    fructose_g = nutriments.get("fructose_100g", 0) or 0
    if carbs_g > 25:
        score += 1; drivers.append("carbohydrates")
    if sugars_g > 8:
        score += 1; drivers.append("sugars")
    if fructose_g > 5:
        score += 1; drivers.append("fructose")

    # Fats & alcohol
    fat_g = nutriments.get("fat_100g", 0) or 0
    satfat_g = nutriments.get("saturated-fat_100g", nutriments.get("saturatedfat_100g", 0)) or 0
    alcohol_g = nutriments.get("alcohol_100g", 0) or 0
    if satfat_g > 4:
        score += 1; drivers.append("saturated fat")
    if satfat_g > 10:
        score += 1
    if fat_g > 20:
        score += 1; drivers.append("total fat")
    if alcohol_g > 1:
        score += 1; drivers.append("alcohol")
    if alcohol_g > 2:
        score += 1

    # Moderate‑purine veg flags: add warning, especially if processed
    if tokens & MODERATE_VEG:
        score += 1; drivers.append("moderate‑purine vegetables")
        if is_processed:
            score += 1; drivers.append("processed vegetable product")

    # Tomato: tiny +1, +2 if processed concentrate/paste context inferred
    if tokens & TOMATO_TOKENS:
        score += 1; drivers.append("tomatoes (possible trigger)")
        if is_processed:
            score += 1; drivers.append("processed tomato")

    # Protective
    vitamin_c_mg = nutriments.get("vitamin-c_100g", 0) or 0
    omega3_g = nutriments.get("omega-3-fat_100g", 0) or 0
    if vitamin_c_mg > 20:
        score -= 1; drivers.append("vitamin C")
    if omega3_g > 0.5:
        score -= 1; drivers.append("omega-3")

    if score >= 2:
        label = "🔥 Pro-inflammatory"
    elif score <= -2:
        label = "🌿 Anti-inflammatory"
    else:
        label = "⚪ Neutral"

    reasons = ", ".join(dict.fromkeys(drivers)) if drivers else "balanced profile"
    return score, label, reasons

# --- Main product parser ---
def get_product_purine_estimate(barcode):
    product = fetch_off_product(barcode)
    if not product or "product" not in product:
        return None

    data = product.get("product", {}) or {}
    product_name = data.get("product_name", "Unknown")
    categories = data.get("categories_tags", []) or []
    nutriments = data.get("nutriments", {}) or {}
    ingredients_text = data.get("ingredients_text", "") or ""

    # Map category and detect tokens/context
    food_category = _map_category(categories, ingredients_text)
    tokens = set()
    # detect moderate veg & tomato & organ cues from ingredients
    lower_ing = (ingredients_text or "").lower()
    for token in MODERATE_VEG:
        if token in lower_ing:
            tokens.add(token)
    for token in TOMATO_TOKENS:
        if token in lower_ing:
            tokens.add(token)
    for c in ["liver","pâté","pate","offal","giblets","sweetbread","kidney","heart","tongue"]:
        if c in lower_ing:
            tokens.add(c)

    is_processed = _infer_processed(categories, ingredients_text)

    # Compute adjusted protein and nucleotides
    adjusted_protein_g, nucleotides_g = get_adjusted_protein_and_nucleotides(nutriments)

    # Estimate purines
    estimated_purine_mg = estimate_purines(food_category, adjusted_protein_g, nucleotides_g, nutriments, tokens, is_processed)

    # Risk index
    purine_risk = classify_purine_risk(estimated_purine_mg)

    # Inflammation
    infl_score, inflammation_impact, infl_reasons = classify_inflammation(food_category, nutriments, tokens, is_processed)

    # Why message
    why_parts = []
    if food_category in ("organ meats", "seafood", "red meat", "poultry"):
        why_parts.append(f"{food_category} baseline")
    elif food_category == "sauces/condiments":
        why_parts.append("plant-based sauce (low purine)")

    carbs_g = nutriments.get("carbohydrates_100g", 0) or 0
    sugars_g = nutriments.get("sugars_100g", nutriments.get("sugar_100g", 0)) or 0
    fructose_g = nutriments.get("fructose_100g", 0) or 0
    fat_g = nutriments.get("fat_100g", 0) or 0
    satfat_g = nutriments.get("saturated-fat_100g", nutriments.get("saturatedfat_100g", 0)) or 0

    if carbs_g > 25 or sugars_g > 8 or fructose_g > 5:
        why_parts.append("carbohydrates/sugars influence")
    if fat_g > 20 or satfat_g > 4:
        why_parts.append("fat/saturated fat influence")
    if tokens & MODERATE_VEG:
        why_parts.append("moderate‑purine vegetables present (moderation advised)")
    if tokens & TOMATO_TOKENS:
        why_parts.append("tomatoes present (possible trigger)")
    if is_processed:
        why_parts.append("processed product context")

    if not why_parts:
        why_parts.append("no animal purine sources detected; plant or mixed composition")

    why_message = "; ".join(dict.fromkeys(why_parts))

    return {
        "product_name": product_name,
        "food_category": food_category,
        "estimated_purine_content_mg_100g": estimated_purine_mg,
        "purine_risk": purine_risk,
        "inflammatory_index": inflammation_impact,
        "why": why_message
    }

# --- CLI ---
if __name__ == "__main__":
    barcode = input("Enter product barcode: ").strip()
    result = get_product_purine_estimate(barcode)

    if result:
        print(f"\nProduct: {result['product_name']}")
        print(f"Food Category: {result['food_category']}")
        print(f"Estimated Purine Content: {result['estimated_purine_content_mg_100g']} mg/100g")
        print(f"Purine Risk: {result['purine_risk']}")
        print(f"Inflammatory Index: {result['inflammatory_index']}")
        print(f"Why: {result['why']}")
    else:
        print("Could not estimate purine content for this product.")
