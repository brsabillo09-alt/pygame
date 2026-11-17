import pygame
import random
import math
import sys

# --- INITIALIZATION ---
pygame.init()
WIDTH, HEIGHT = 1000, 680
screen = pygame.display.set_mode((WIDTH, HEIGHT))
pygame.display.set_caption("The Pressure Cooker Kitchen Rush — Thermodynamics Edition")
clock = pygame.time.Clock()

# --- FONTS & COLORS ---
FONT_SM = pygame.font.SysFont("consolas", 15)
FONT_MD = pygame.font.SysFont("consolas", 18, bold=True)
FONT_LG = pygame.font.SysFont("consolas", 24, bold=True)
FONT_XL = pygame.font.SysFont("consolas", 36, bold=True)

BG_DARK = (26, 28, 36)
PANEL_BG = (38, 42, 54)
POT_METAL = (110, 118, 135)
POT_INTERIOR = (18, 22, 30)
ACCENT_BLUE = (72, 180, 240)
ACCENT_ORANGE = (255, 130, 50)
ACCENT_RED = (235, 65, 75)
ACCENT_GREEN = (60, 210, 120)
TEXT_LIGHT = (235, 240, 245)
TEXT_MUTED = (150, 160, 175)

# --- THERMODYNAMIC CONSTANTS (Ideal Gas Law: PV = nRT) ---
R_CONST = 0.0821       # L·atm / (mol·K)
POT_VOLUME = 6.0       # Fixed volume (6.0 Liters)
AMBIENT_TEMP = 293.15  # 20°C in Kelvin
ALTITUDE_P = 0.70      # High-altitude ambient pressure (0.70 atm ~ 3,000m elevation)
ALTITUDE_BOIL_K = 363.15 # Water boils at ~90°C (363.15 K) at 0.70 atm
MAX_SAFE_PRESSURE = 2.60 # Burst limit in atm

# --- RECIPES ---
RECIPES = [
    {"name": "Alpine Ramen", "target_p": 1.35, "tol": 0.15, "cook_req": 5.0, "time_limit": 35},
    {"name": "Summit Beef Stew", "target_p": 1.85, "tol": 0.15, "cook_req": 6.5, "time_limit": 40},
    {"name": "Glacier Bone Broth", "target_p": 2.25, "tol": 0.12, "cook_req": 7.5, "time_limit": 45},
    {"name": "Highland Risotto", "target_p": 1.55, "tol": 0.15, "cook_req": 5.5, "time_limit": 35},
]

# --- PARTICLE CLASS (Water Vapor Molecules) ---
class VaporParticle:
    def __init__(self, bounds_rect):
        self.bounds = bounds_rect
        self.radius = 4
        self.x = random.uniform(bounds_rect.left + 10, bounds_rect.right - 10)
        self.y = random.uniform(bounds_rect.top + 10, bounds_rect.bottom - 10)
        angle = random.uniform(0, 2 * math.pi)
        self.vx = math.cos(angle)
        self.vy = math.sin(angle)

    def update(self, dt, temp_k):
        # Root-mean-square speed scales with sqrt(Temperature)
        speed_scale = math.sqrt(temp_k / 293.15) * 180.0
        self.x += self.vx * speed_scale * dt
        self.y += self.vy * speed_scale * dt

        # Wall collisions (elastic)
        if self.x - self.radius < self.bounds.left:
            self.x = self.bounds.left + self.radius
            self.vx *= -1
        elif self.x + self.radius > self.bounds.right:
            self.x = self.bounds.right - self.radius
            self.vx *= -1

        if self.y - self.radius < self.bounds.top:
            self.y = self.bounds.top + self.radius
            self.vy *= -1
            return True  # Hit lid (visual impulse)
        elif self.y + self.radius > self.bounds.bottom:
            self.y = self.bounds.bottom - self.radius
            self.vy *= -1
        return False

    def draw(self, surface, temp_k):
        # Color shifts from cool cyan to hot orange-red with temperature
        heat_ratio = max(0.0, min(1.0, (temp_k - 293.15) / (450.0 - 293.15)))
        r = int(72 + (255 - 72) * heat_ratio)
        g = int(180 - 100 * heat_ratio)
        b = int(240 - 190 * heat_ratio)
        pygame.draw.circle(surface, (r, g, b), (int(self.x), int(self.y)), self.radius)

# --- MAIN GAME ENGINE ---
class PressureCookerGame:
    def __init__(self):
        self.pot_rect = pygame.Rect(330, 210, 340, 260)
        self.reset_game()

    def reset_game(self):
        self.score = 0
        self.dishes_served = 0
        self.game_over = False
        self.game_over_reason = ""
        
        # Thermodynamic State Variables
        self.temperature_k = AMBIENT_TEMP   # Kelvin (T)
        self.moles_gas = 0.175              # Moles of air/vapor (n) -> P = nRT/V ≈ 0.70 atm initially
        self.pressure_atm = ALTITUDE_P      # Pressure (P)
        self.burner_level = 0               # 0 (Off), 1 (Low), 2 (Med), 3 (High)
        self.valve_open = False
        self.xray_window = True             # Toggle cross-section particle view
        
        # Lid vibration & physics feedback
        self.lid_shake = 0.0
        self.accuracy_samples = []
        
        # Particles
        self.particles = [VaporParticle(self.pot_rect) for _ in range(35)]
        
        # Order Management
        self.spawn_order()

    def spawn_order(self):
        recipe = random.choice(RECIPES)
        self.current_order = recipe.copy()
        self.order_timer = recipe["time_limit"]
        self.cook_progress = 0.0
        self.accuracy_samples = []

    def sync_particles(self):
        # Keep visual particle count proportional to moles (n)
        target_count = max(12, min(140, int(self.moles_gas * 200)))
        while len(self.particles) < target_count:
            self.particles.append(VaporParticle(self.pot_rect))
        while len(self.particles) > target_count:
            self.particles.pop()

    def update(self, dt):
        if self.game_over:
            return

        # 1. HEAT TRANSFER (Burner adds energy; ambient mountain air cools pot)
        heat_input = self.burner_level * 26.0  # Kelvin/sec heating rate
        ambient_loss = (self.temperature_k - AMBIENT_TEMP) * 0.14
        self.temperature_k += (heat_input - ambient_loss) * dt
        self.temperature_k = max(AMBIENT_TEMP, self.temperature_k)

        # 2. PHASE CHANGE (Water boiling into steam above altitude boiling point)
        if self.temperature_k > ALTITUDE_BOIL_K and not self.valve_open:
            # Vaporization increases moles of gas (n) inside sealed chamber
            vapor_rate = (self.temperature_k - ALTITUDE_BOIL_K) * 0.0022
            self.moles_gas += vapor_rate * dt

        # 3. STEAM VALVE RELEASE (Vents moles of gas, dropping pressure & cooling slightly)
        if self.valve_open and self.pressure_atm > ALTITUDE_P:
            vent_rate = (self.pressure_atm - ALTITUDE_P) * 0.35
            self.moles_gas = max(0.175, self.moles_gas - vent_rate * dt)
            # Adiabatic expansion cooling effect
            self.temperature_k -= 18.0 * dt

        # 4. IDEAL & GAY-LUSSAC'S LAW CALCULATION: P = (n * R * T) / V
        self.pressure_atm = (self.moles_gas * R_CONST * self.temperature_k) / POT_VOLUME
        if self.pressure_atm < ALTITUDE_P:
            self.pressure_atm = ALTITUDE_P

        # 5. OVER-PRESSURIZATION CHECK (Safety Valve Burst)
        if self.pressure_atm >= MAX_SAFE_PRESSURE:
            self.game_over = True
            self.game_over_reason = f"SAFETY VALVE RUPTURED! Pressure hit {self.pressure_atm:.2f} atm (Max: {MAX_SAFE_PRESSURE} atm)."
            return

        # 6. PARTICLE SIMULATION
        self.sync_particles()
        lid_hits = 0
        for p in self.particles:
            if p.update(dt, self.temperature_k):
                lid_hits += 1
        self.lid_shake = min(5.0, (self.pressure_atm - ALTITUDE_P) * 2.2)

        # 7. COOKING PROGRESS & MATHEMATICAL ACCURACY SCORING
        self.order_timer -= dt
        if self.order_timer <= 0:
            self.game_over = True
            self.game_over_reason = f"TIME EXPIRED! Customers walked out on {self.current_order['name']}."
            return

        target_p = self.current_order["target_p"]
        tol = self.current_order["tol"]
        p_diff = abs(self.pressure_atm - target_p)

        # Food only cooks efficiently when pressure raises boiling point sufficiently
        if p_diff <= tol:
            self.cook_progress += dt
            # Verify P/T ratio matches theoretical Gay-Lussac constant (nR/V)
            theoretical_ratio = (self.moles_gas * R_CONST) / POT_VOLUME
            actual_ratio = self.pressure_atm / self.temperature_k
            ratio_error = abs(theoretical_ratio - actual_ratio) / theoretical_ratio
            precision = max(0.0, 1.0 - (p_diff / tol) * 0.4 - ratio_error)
            self.accuracy_samples.append(precision)

            if self.cook_progress >= self.current_order["cook_req"]:
                self.complete_order()

    def complete_order(self):
        avg_acc = sum(self.accuracy_samples) / max(1, len(self.accuracy_samples))
        time_bonus = int(self.order_timer * 10)
        thermo_bonus = int(avg_acc * 250)
        self.score += 200 + time_bonus + thermo_bonus
        self.dishes_served += 1
        self.spawn_order()

    def draw(self):
        screen.fill(BG_DARK)

        # --- HEADER ---
        title = FONT_LG.render("THE PRESSURE COOKER KITCHEN RUSH", True, TEXT_LIGHT)
        subtitle = FONT_SM.render("Elevation: 3,000m (Ambient P = 0.70 atm | Water Boils at 90°C / 363 K)", True, ACCENT_BLUE)
        screen.blit(title, (25, 18))
        screen.blit(subtitle, (25, 48))

        score_txt = FONT_LG.render(f"SCORE: {self.score}  |  DISHES: {self.dishes_served}", True, ACCENT_GREEN)
        screen.blit(score_txt, (WIDTH - score_txt.get_width() - 25, 22))

        # --- LEFT PANEL: ORDER & RECIPE TARGET ---
        pygame.draw.rect(screen, PANEL_BG, (25, 85, 275, 260), border_radius=10)
        screen.blit(FONT_MD.render("ACTIVE ORDER", True, ACCENT_ORANGE), (40, 100))
        screen.blit(FONT_LG.render(self.current_order["name"], True, TEXT_LIGHT), (40, 130))

        t_p = self.current_order["target_p"]
        tol = self.current_order["tol"]
        # Calculate required temperature at current moles to hit target P (T = PV / nR)
        req_temp_k = (t_p * POT_VOLUME) / (self.moles_gas * R_CONST)
        
        screen.blit(FONT_SM.render(f"Target Pressure: {t_p:.2f} ±{tol:.2f} atm", True, TEXT_LIGHT), (40, 170))
        screen.blit(FONT_SM.render(f"Est. Target Temp: {req_temp_k:.0f} K ({req_temp_k-273.15:.0f}°C)", True, TEXT_MUTED), (40, 192))
        screen.blit(FONT_SM.render(f"Time Left: {max(0, self.order_timer):.1f}s", True, ACCENT_RED if self.order_timer < 10 else TEXT_LIGHT), (40, 220))

        # Cook Progress Bar
        screen.blit(FONT_SM.render("Cooking Progress:", True, TEXT_LIGHT), (40, 255))
        pygame.draw.rect(screen, BG_DARK, (40, 278, 240, 22), border_radius=5)
        prog_ratio = min(1.0, self.cook_progress / self.current_order["cook_req"])
        pygame.draw.rect(screen, ACCENT_GREEN, (40, 278, int(240 * prog_ratio), 22), border_radius=5)

        # --- LEFT BOTTOM PANEL: CONTROLS ---
        pygame.draw.rect(screen, PANEL_BG, (25, 365, 275, 285), border_radius=10)
        screen.blit(FONT_MD.render("KITCHEN CONTROLS", True, ACCENT_BLUE), (40, 380))
        controls = [
            "[UP / DOWN]  Adjust Burner Heat",
            "[SPACE]      Hold Steam Valve",
            "[TAB]        Toggle X-Ray Pot",
            "[R]          Restart Shift",
            "",
            "TIP: High altitude lowers",
            "boiling point to 90°C. Seal",
            "pot & raise P to cook fast!"
        ]
        for idx, line in enumerate(controls):
            color = TEXT_LIGHT if idx < 4 else TEXT_MUTED
            screen.blit(FONT_SM.render(line, True, color), (40, 415 + idx * 26))

        # --- CENTER: PRESSURE COOKER & PARTICLE SIMULATION ---
        shake_x = random.uniform(-self.lid_shake, self.lid_shake)
        shake_y = random.uniform(-self.lid_shake, self.lid_shake)

        # Burner Flame Visuals
        if self.burner_level > 0:
            flame_color = [ACCENT_BLUE, ACCENT_ORANGE, ACCENT_RED][self.burner_level - 1]
            for i in range(5):
                fx = self.pot_rect.left + 40 + i * 65
                fy = self.pot_rect.bottom + 18
                pygame.draw.polygon(screen, flame_color, [(fx - 12, fy + 18), (fx + 12, fy + 18), (fx, fy - self.burner_level * 6)])

        # Pot Body
        outer_pot = self.pot_rect.inflate(24, 24)
        pygame.draw.rect(screen, POT_METAL, outer_pot, border_radius=14)
        pygame.draw.rect(screen, POT_INTERIOR, self.pot_rect, border_radius=8)

        # Heavy Locking Lid
        lid_rect = pygame.Rect(outer_pot.left - 8 + shake_x, outer_pot.top - 22 + shake_y, outer_pot.width + 16, 24)
        pygame.draw.rect(screen, (160, 170, 185), lid_rect, border_radius=6)

        # Steam Release Valve on Lid
        valve_x = lid_rect.right - 55
        valve_color = ACCENT_RED if self.valve_open else (80, 90, 105)
        pygame.draw.rect(screen, valve_color, (valve_x, lid_rect.top - 16, 22, 16), border_radius=3)
        if self.valve_open and self.pressure_atm > ALTITUDE_P + 0.05:
            # Venting steam particles effect
            for _ in range(8):
                sx = valve_x + 11 + random.uniform(-18, 18)
                sy = lid_rect.top - 20 - random.uniform(5, 55)
                pygame.draw.circle(screen, (220, 235, 245), (int(sx), int(sy)), random.randint(3, 7))

        # Interior View: Either X-Ray Particle Simulation or Opaque Steel Wall
        if self.xray_window:
            for p in self.particles:
                p.draw(screen, self.temperature_k)
            lbl = FONT_SM.render(f"X-RAY VIEW: {len(self.particles)} H2O Vapor Molecules", True, TEXT_MUTED)
            screen.blit(lbl, (self.pot_rect.centerx - lbl.get_width() // 2, self.pot_rect.bottom - 24))
        else:
            pygame.draw.rect(screen, (85, 92, 108), self.pot_rect, border_radius=8)
            lbl = FONT_MD.render("SEALED STAINLESS STEEL POT", True, TEXT_LIGHT)
            sub = FONT_SM.render("Press [TAB] for Particle X-Ray", True, TEXT_MUTED)
            screen.blit(lbl, (self.pot_rect.centerx - lbl.get_width() // 2, self.pot_rect.centery - 15))
            screen.blit(sub, (self.pot_rect.centerx - sub.get_width() // 2, self.pot_rect.centery + 12))

        # Burner Level Indicator below pot
        burner_names = ["OFF", "LOW", "MEDIUM", "HIGH"]
        b_txt = FONT_MD.render(f"BURNER: {burner_names[self.burner_level]}", True, ACCENT_ORANGE if self.burner_level else TEXT_MUTED)
        screen.blit(b_txt, (self.pot_rect.centerx - b_txt.get_width() // 2, self.pot_rect.bottom + 48))

        # --- RIGHT PANEL: THERMODYNAMIC TELEMETRY & GAS LAW MATH ---
        pygame.draw.rect(screen, PANEL_BG, (705, 85, 270, 565), border_radius=10)
        screen.blit(FONT_MD.render("THERMODYNAMICS LAB", True, ACCENT_BLUE), (720, 10
