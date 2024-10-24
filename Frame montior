#!/bin/bash

# ceremony-monitor-install.sh
echo "Installing Ceremony Monitor..."

# Check if running as root
if [ "$EUID" -ne 0 ]; then 
    echo "Please run as root (use sudo)"
    exit 1
fi

# Create python script
cat > /usr/local/bin/ceremony_monitor.py << 'EOL'
import time
import subprocess
import json
import logging
import signal
import sys
from datetime import datetime, timedelta

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('/var/log/ceremony_monitor.log'),
        logging.StreamHandler()
    ]
)

class CeremonyMonitor:
    def __init__(self, service_name):
        self.service_name = service_name
        self.last_frame_number = None
        self.last_progress_time = None
        self.CHECK_INTERVAL = timedelta(seconds=30)  # Check every 30 seconds
        self.STALL_THRESHOLD = timedelta(minutes=10)  # Wait 10 minutes before restarting
        self.RESTART_COOLDOWN = timedelta(minutes=15)
        self.running = True

    def signal_handler(self, signum, frame):
        """Handle shutdown signals gracefully"""
        logging.info(f"Received signal {signum}. Shutting down...")
        self.running = False

    def get_latest_frame_number(self):
        """Extract the latest frame number from journalctl logs."""
        try:
            cmd = f"journalctl -u {self.service_name}.service --no-hostname -o cat -n 100"
            result = subprocess.run(cmd.split(), capture_output=True, text=True)
            
            if result.returncode != 0:
                logging.error(f"Failed to read journalctl: {result.stderr}")
                return None
                
            latest_frame = None
            for line in reversed(result.stdout.splitlines()):
                try:
                    log_entry = json.loads(line.strip())
                    if "frame_number" in log_entry:
                        latest_frame = log_entry["frame_number"]
                        break
                except (json.JSONDecodeError, ValueError, KeyError) as e:
                    continue
            
            if latest_frame is not None:
                logging.info(f"Latest frame number found: {latest_frame}")
            return latest_frame

        except Exception as e:
            logging.error(f"Error reading logs: {e}")
            return None

    def restart_service(self):
        """Restart the ceremony client service using systemctl."""
        try:
            logging.warning(f"Initiating {self.service_name} restart...")
            
            subprocess.run(['systemctl', 'stop', self.service_name], check=True)
            logging.info("Service stopped")
            
            time.sleep(5)
            
            subprocess.run(['systemctl', 'start', self.service_name], check=True)
            logging.info("Service started")
            
            status = subprocess.run(['systemctl', 'is-active', self.service_name], 
                                  capture_output=True, text=True).stdout.strip()
            if status == 'active':
                logging.info("Service restart completed successfully")
            else:
                logging.error(f"Service status after restart: {status}")
                
        except subprocess.CalledProcessError as e:
            logging.error(f"Failed to restart service: {e}")

    def monitor(self):
        """Main monitoring loop."""
        logging.info(f"Starting ceremony client monitor for service: {self.service_name}")
        
        signal.signal(signal.SIGTERM, self.signal_handler)
        signal.signal(signal.SIGINT, self.signal_handler)
        
        while self.running:
            try:
                current_frame = self.get_latest_frame_number()
                current_time = datetime.now()

                if current_frame is not None:
                    if self.last_frame_number is None:
                        self.last_frame_number = current_frame
                        self.last_progress_time = current_time
                        logging.info(f"Initial frame number: {current_frame}")
                    elif current_frame > self.last_frame_number:
                        logging.info(f"Frame progress: {self.last_frame_number} -> {current_frame}")
                        self.last_frame_number = current_frame
                        self.last_progress_time = current_time
                    else:
                        stall_time = current_time - self.last_progress_time
                        if stall_time >= self.STALL_THRESHOLD:
                            logging.warning(f"Frame number stalled at {current_frame} for {stall_time.total_seconds()/60:.1f} minutes")
                            self.restart_service()
                            
                            logging.info(f"Entering cooldown period for {self.RESTART_COOLDOWN.total_seconds()/60} minutes")
                            time.sleep(self.RESTART_COOLDOWN.total_seconds())
                            
                            self.last_frame_number = None
                            self.last_progress_time = None
                        else:
                            logging.info(f"No progress yet, but only stalled for {stall_time.total_seconds()/60:.1f} minutes")

                for _ in range(int(self.CHECK_INTERVAL.total_seconds())):
                    if not self.running:
                        break
                    time.sleep(1)

            except Exception as e:
                logging.error(f"Monitor loop error: {e}")
                if self.running:
                    time.sleep(self.CHECK_INTERVAL.total_seconds())

        logging.info("Monitor shutting down gracefully")

if __name__ == "__main__":
    SERVICE_NAME = "ceremonyclient"
    
    monitor = CeremonyMonitor(SERVICE_NAME)
    monitor.monitor()
    sys.exit(0)
EOL

# Create systemd service file
cat > /lib/systemd/system/ceremony-monitor.service << 'EOL'
[Unit]
Description=Ceremony Client Monitor Service
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/python3 /usr/local/bin/ceremony_monitor.py
User=root
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOL

# Set permissions
chmod 755 /usr/local/bin/ceremony_monitor.py
chown root:root /usr/local/bin/ceremony_monitor.py
touch /var/log/ceremony_monitor.log
chmod 644 /var/log/ceremony_monitor.log
chown root:root /var/log/ceremony_monitor.log

# Reload systemd and start service
systemctl daemon-reload
systemctl enable ceremony-monitor
systemctl start ceremony-monitor

echo "Installation completed!"
echo "Monitor service has been installed and started."
echo ""
echo "You can:"
echo "- Check status: systemctl status ceremony-monitor"
echo "- View logs: journalctl -u ceremony-monitor -f"
echo "- View monitor log: tail -f /var/log/ceremony_monitor.log"
echo ""
echo "The monitor will:"
echo "- Check frame numbers every 30 seconds"
echo "- Restart ceremonyclient if frames don't increase for 10 minutes"
echo "- Wait 15 minutes after a restart before resuming monitoring"
EOL
