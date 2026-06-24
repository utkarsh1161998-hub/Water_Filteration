import time
import logging
import random
from datetime import datetime
from typing import Dict, Any

# Configure structured logging
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
    handlers=[logging.StreamHandler()]
)

class WaterFiltrationSystem:
    def __init__(self):
        # System States: 'FILTRATION', 'BACKWASH', 'RINSE', 'EMERGENCY_STOP'
        self.state = "FILTRATION"
        self.is_running = True
        
        # Sensor Thresholds
        self.MAX_PRESSURE_DROP = 15.0  # PSI (Delta P trigger for backwash)
        self.MIN_PH = 6.5
        self.MAX_PH = 8.5
        self.MAX_TDS = 500  # PPM (Total Dissolved Solids)
        
        # Actuator States (Simulated Relays/Valves)
        self.valves = {
            "inlet": "OPEN",
            "outlet": "OPEN",
            "backwash_drain": "CLOSED",
            "rinse_drain": "CLOSED"
        }
        self.pump_speed = 100  # Percentage

    def read_sensors(self) -> Dict[str, float]:
        """
        Simulates reading telemetry from hardware sensors.
        In production, replace this with Modbus, I2C, or MQTT fetches.
        """
        if self.state == "FILTRATION":
            # Simulate gradual filter clogging over time
            p_in = round(random.uniform(40.0, 45.0), 2)
            p_out = round(random.uniform(28.0, 35.0), 2)  # Lower p_out = dirtier filter
        else:
            p_in, p_out = 30.0, 29.0

        return {
            "pressure_in": p_in,
            "pressure_out": p_out,
            "delta_p": round(p_in - p_out, 2),
            "ph": round(random.uniform(6.2, 8.8), 2),  # Expanded to simulate fluctuations
            "tds": random.randint(100, 600)
        }

    def set_valves(self, inlet: str, outlet: str, bw_drain: str, rinse: str):
        """Helper to command physical or simulated actuators."""
        self.valves["inlet"] = inlet
        self.valves["outlet"] = outlet
        self.valves["backwash_drain"] = bw_drain
        self.valves["rinse_drain"] = rinse
        logging.info(f"Valves Updated -> Inlet: {inlet} | Outlet: {outlet} | BW Drain: {bw_drain} | Rinse Drain: {rinse}")

    def execute_backwash_cycle(self):
        """Automated multi-stage cleaning cycle."""
        logging.warning(" Critical filter loading detected. Initiating automated backwash sequence!")
        
        # Step 1: Shut down filtration, open backwash drain
        self.state = "BACKWASH"
        self.pump_speed = 50
        self.set_valves(inlet="CLOSED", outlet="CLOSED", bw_drain="OPEN", rinse="CLOSED")
        logging.info(" Backwashing filter media (Reversed flow)...")
        time.sleep(4)  # Simulated duration

        # Step 2: Rinse cycle to settle media and clear dirty residual water
        self.state = "RINSE"
        self.set_valves(inlet="OPEN", outlet="CLOSED", bw_drain="CLOSED", rinse="OPEN")
        logging.info(" Rinsing filter media to drain...")
        time.sleep(2)  # Simulated duration

        # Step 3: Return to standard filtration
        logging.info(" Cleaning cycle complete. Returning to normal filtration.")
        self.set_valves(inlet="OPEN", outlet="OPEN", bw_drain="CLOSED", rinse="CLOSED")
        self.pump_speed = 100
        self.state = "FILTRATION"

    def safety_interlock_check(self, telemetry: Dict[str, float]) -> bool:
        """Evaluates system safety limits to prevent hardware damage or toxic output."""
        if telemetry["ph"] < self.MIN_PH or telemetry["ph"] > self.MAX_PH:
            logging.critical(f" Out-of-bounds pH level detected: {telemetry['ph']}!")
            return False
        if telemetry["tds"] > self.MAX_TDS:
            logging.critical(f" Contaminated water detected! TDS level: {telemetry['tds']} PPM")
            return False
        return True

    def run_control_loop(self):
        """Main Scada/PLC style control loop execution."""
        logging.info(" Water Filtration Automation System Online.")
        
        try:
            while self.is_running:
                # 1. Gather Data
                telemetry = self.read_sensors()
                logging.info(f"State: [{self.state}] | ΔP: {telemetry['delta_p']} PSI | pH: {telemetry['ph']} | TDS: {telemetry['tds']} PPM")

                # 2. Safety Evaluation
                if not self.safety_interlock_check(telemetry):
                    self.state = "EMERGENCY_STOP"
                    self.set_valves("CLOSED", "CLOSED", "CLOSED", "CLOSED")
                    self.pump_speed = 0
                    logging.critical(" Emergency Stop Engaged. System isolated. Awaiting manual override.")
                    break

                # 3. Process Control Logic
                if self.state == "FILTRATION":
                    if telemetry["delta_p"] >= self.MAX_PRESSURE_DROP:
                        self.execute_backwash_cycle()

                time.sleep(1.5)  # Loop frequency (1.5s intervals)

        except KeyboardInterrupt:
            logging.info("Stopping automation engine gracefully via user request.")
        finally:
            self.is_running = False

# --- Execution Block ---
if __name__ == "__main__":
    automation_system = WaterFiltrationSystem()
    automation_system.run_control_loop()
