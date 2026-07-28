# voice.py
a little voice assistant i made there will be bugs




the code    |
            v





            #!/usr/bin/env python3
import asyncio
import os
import subprocess
import speech_recognition as sr
import aiohttp
import psutil

class VoiceAssistant:
    def __init__(self, wake_word="system"):
        self.wake_word = wake_word.lower()
        self.recognizer = sr.Recognizer()
        # Adjust energy threshold for microphone sensitivity
        self.recognizer.energy_threshold = 300
        self.recognizer.dynamic_energy_threshold = True
        self.cava_process = None

    async def speak(self, text: str):
        """Asynchronous text-to-speech using espeak-ng."""
        print(f"Assistant: {text}")
        await asyncio.to_thread(
            subprocess.run,
            ["espeak-ng", "-s", "160", text],
            stdout=subprocess.DEVNULL,
            stderr=subprocess.DEVNULL
        )

    async def listen(self) -> str:
        """Non-blocking audio capture using speech_recognition in a background thread."""
        def _capture():
            with sr.Microphone() as source:
                self.recognizer.adjust_for_ambient_noise(source, duration=0.5)
                print("Listening...")
                try:
                    audio = self.recognizer.listen(source, timeout=5, phrase_time_limit=5)
                    text = self.recognizer.recognize_google(audio)
                    return text.lower()
                except (sr.WaitTimeoutError, sr.UnknownValueError):
                    return ""
                except sr.RequestError as e:
                    print(f"Speech recognition error: {e}")
                    return ""

        return await asyncio.to_thread(_capture)

    # ---------------------------------------------------------
    # System & Hardware Controls
    # ---------------------------------------------------------
    async def get_system_temps(self) -> str:
        """Fetch hardware temperatures using psutil."""
        try:
            temps = psutil.sensors_temperatures()
            if not temps:
                return "Temperature sensors not available."

            for sensor_name in ["coretemp", "k10temp", "zenpower", "cpu_thermal"]:
                if sensor_name in temps:
                    cpu_temp = temps[sensor_name][0].current
                    return f"CPU temperature is {cpu_temp:.1f} degrees Celsius."

            first_sensor = list(temps.keys())[0]
            temp = temps[first_sensor][0].current
            return f"System temperature is {temp:.1f} degrees Celsius."
        except Exception:
            return "Failed to read temperature sensors."

    async def set_cpu_governor(self, mode: str):
        """Change CPU governor using cpupower."""
        try:
            cmd = ["sudo", "cpupower", "frequency-set", "-g", mode]
            subprocess.run(cmd, check=True)
            await self.speak(f"CPU governor set to {mode}.")
        except Exception:
            await self.speak("Failed to change CPU governor. Check sudo permissions.")

    async def fetch_weather(self, city: str = "") -> str:
        """Fetch current weather via wttr.in asynchronously."""
        url = f"https://wttr.in/{city}?format=3"
        try:
            async with aiohttp.ClientSession() as session:
                async with session.get(url, timeout=5) as response:
                    if response.status == 200:
                        data = await response.text()
                        return data.strip()
                    return "Could not retrieve weather data."
        except Exception:
            return "Weather service is currently unavailable."

    # ---------------------------------------------------------
    # Hyprland & Visuals Integration
    # ---------------------------------------------------------
    async def hyprctl_dispatch(self, command: str):
        """Send IPC commands to Hyprland window manager."""
        try:
            subprocess.run(["hyprctl", "dispatch", command])
        except Exception as e:
            print(f"Hyprland dispatch error: {e}")

    async def toggle_cava(self):
        """Toggle CAVA audio visualizer terminal window."""
        if self.cava_process is None or self.cava_process.poll() is not None:
            self.cava_process = subprocess.Popen(["kitty", "--class", "cava_float", "-e", "cava"])
            await self.speak("Visualizer enabled.")
        else:
            self.cava_process.terminate()
            self.cava_process = None
            await self.speak("Visualizer disabled.")

    # ---------------------------------------------------------
    # Command Parser
    # ---------------------------------------------------------
    async def execute_command(self, text: str):
        # --- Audio & Volume ---
        if "volume up" in text:
            subprocess.run(["wpctl", "set-volume", "@DEFAULT_AUDIO_SINK@", "5%+"])
            await self.speak("Volume increased.")
            
        elif "volume down" in text:
            subprocess.run(["wpctl", "set-volume", "@DEFAULT_AUDIO_SINK@", "5%-"])
            await self.speak("Volume decreased.")

        # --- Terranova Command ---
        elif "terranova" in text or "terra nova" in text:
            await self.speak("Playing Terranova on YouTube.")
            yt_url = "https://www.youtube.com/watch?v=4tys_2aPvLM"
            await self.hyprctl_dispatch(f"exec xdg-open '{yt_url}'")
            await asyncio.sleep(2)
            subprocess.run(["playerctl", "play"])

        # --- Media Controls ---
        elif "play" in text or "pause" in text:
            subprocess.run(["playerctl", "play-pause"])
            await self.speak("Toggled media.")

        elif "next track" in text or "skip" in text:
            subprocess.run(["playerctl", "next"])

        # --- Visualizer & Window Management ---
        elif "visualizer" in text or "cava" in text:
            await self.toggle_cava()

        elif "workspace" in text:
            words = text.split()
            if len(words) > 1 and words[-1].isdigit():
                ws_num = words[-1]
                await self.hyprctl_dispatch(f"workspace {ws_num}")
                await self.speak(f"Switched to workspace {ws_num}.")

        # --- Hardware & Performance ---
        elif "performance mode" in text:
            await self.set_cpu_governor("performance")

        elif "power save mode" in text or "eco mode" in text:
            await self.set_cpu_governor("powersave")

        elif "temperature" in text or "temp" in text or "how hot" in text:
            temp_msg = await self.get_system_temps()
            await self.speak(temp_msg)

        elif "system status" in text or "cpu" in text:
            try:
                load = os.getloadavg()[0]
                temp_msg = await self.get_system_temps()
                await self.speak(f"System load is {load:.2f}. {temp_msg}")
            except Exception:
                await self.speak("Unable to fetch system status.")

        # --- Weather ---
        elif "weather" in text:
            weather_info = await self.fetch_weather()
            await self.speak(f"Current weather: {weather_info}")

        # --- Easter Eggs ---
        elif "protocol 3" in text:
            await self.speak("Protocol 3: Protect the Pilot.")
        
        else:
            await self.speak("Command not recognized.")

    # ---------------------------------------------------------
    # Main Loop
    # ---------------------------------------------------------
    async def run(self):
        """Main asynchronous execution loop."""
        await self.speak("Voice assistant online.")
        while True:
            recognized_text = await self.listen()
            if not recognized_text:
                continue

            print(f"Heard: {recognized_text}")

            # Wake word matching
            if self.wake_word in recognized_text:
                command = recognized_text.replace(self.wake_word, "").strip()
                if command:
                    await self.execute_command(command)
                else:
                    await self.speak("Online.")
                    next_text = await self.listen()
                    if next_text:
                        await self.execute_command(next_text)

async def main():
    assistant = VoiceAssistant(wake_word="system")
    await assistant.run()

if __name__ == "__main__":
    try:
        asyncio.run(main())
    except KeyboardInterrupt:
        print("\nAssistant shut down safely.")
