import sys
import requests

# Official/Community API Endpoints
PROTONDB_API_URL = "https://protondb.com{}/reports/summary.json"
AREWEAC_API_URL = "https://areweanticheatyet.com"

def get_protondb_status(app_id):
    """Fetches public ProtonDB tier rating using the Steam AppID."""
    try:
        response = requests.get(PROTONDB_API_URL.format(app_id), timeout=5)
        if response.status_code == 200:
            data = response.json()
            # Returns tier: platinum, gold, silver, bronze, borked
            return data.get("trendingTier", "unknown")
    except requests.RequestException:
        pass
    return "unknown"

def get_anticheat_status(game_name):
    """
    Queries the AreWeAntiCheatYet API for anti-cheat data matching the game name.
    Note: For production, caching this large JSON response locally is highly recommended.
    """
    try:
        response = requests.get(AREWEAC_API_URL, timeout=5)
        if response.status_code == 200:
            games_list = response.json()
            # Search for matches based on game name or slug
            for game in games_list:
                if game_name.lower() in game.get("slug", "").lower() or game_name.lower() in game.get("name", "").lower():
                    ac_mechanism = game.get("anticheat", "none")
                    ac_status = game.get("status", "unknown")
                    return ac_mechanism, ac_status
    except requests.RequestException:
        pass
    return "none", "unknown"

def evaluate_game_pipeline(app_id, game_name):
    print(f"[Orchestrator] Evaluating real-time execution route for: {game_name} ({app_id})...")

    # 1. Dynamic ProtonDB API lookup
    proton_status = get_protondb_status(app_id)
    
    # 2. Dynamic AreWeAntiCheatYet API lookup
    ac_mechanism, ac_status = get_anticheat_status(game_name)

    print(f" -> [ProtonDB]: {proton_status.upper()} | [Anti-Cheat]: {ac_mechanism.upper()} ({ac_status.upper()})")

    # RULE 1: If game runs flawlessly via Proton, deploy natively to host (CachyOS)
    if proton_status in ["platinum", "gold"]:
        return "LAUNCH_PROTON", f"Main Route: Game runs natively via Proton ({proton_status.upper()}). Launching directly on host."

    # RULE 2: If Proton status is failed ('borked') or anti-cheat breaks compatibility
    if proton_status == "borked" or ac_status in ["denied", "unsupported"]:
        
        # Critical Lock: Hostile Kernel-Level Anti-Cheats (e.g., Vanguard, Ricochet)
        # Denies execution if the anti-cheat explicitly bans virtualized/hypervisor environments
        if "vanguard" in ac_mechanism.lower() or ac_status == "denied":
            return "ABORT_LAUNCH", f"CRITICAL SECURITY LOCK: {ac_mechanism.upper()} blocks virtual environments. Launch denied to prevent account ban."
        
        # Game requires a native Windows environment but tolerates hypervisors
        if ac_status == "supported" or ac_status == "running":
            return "LAUNCH_KVM_VFIO", "Advanced Route: Game requires native Windows but tolerates hypervisors. Triggering KVM hardware bindings..."
            
        return "LAUNCH_KVM_VFIO", "Fallback Route: Uncertain compatibility status on Linux. Deploying isolated KVM pipeline for system safety."

    return "LAUNCH_PROTON", "Default Route: Proceeding with standard host launch sequence via Proton."

if __name__ == "__main__":
    # Test execution case:
    # 1172470 = Apex Legends (Gold/Platinum tier, runs well via compatible Easy Anti-Cheat)
    action, message = evaluate_game_pipeline("1172470", "Apex Legends")
    
    print(f"\n[DECISION]: {action}")
    print(f"[REASON]: {message}\n")
