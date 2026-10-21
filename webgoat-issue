# 🛠️ Guide: How to Fix the "Port 9090 is Taken" Error in WebGoat

When you run WebGoat, it launches two tools at the same time: **WebGoat on port 8080** and **WebWolf on port 9090**. If another application on your laptop is already using port 9090, WebGoat will crash on startup. 

Follow this guide to clear the port or bypass the conflict.

---

## 🪟 Method 1: For Windows Users

### Step 1: Open Command Prompt as Administrator
1. Press the **Windows Key** on your keyboard.
2. Type **`cmd`**.
3. Right-click on **Command Prompt** and select **Run as administrator**.

### Step 2: Find the Process Using Port 9090
Type the following command and press **Enter**:
```cmd
netstat -ano | findstr :9090
```
You will see an output line that looks like this:
```text
TCP    0.0.0.0:9090    0.0.0.0:0    LISTENING    4312
```
*The number at the very end (e.g., **`4312`**) is the **PID (Process ID)** hogging the port. Look at your own terminal output and note down your specific PID number.*

### Step 3: Kill the Hogging Process
Type the following command (replace `4312` with the **exact PID number** you found in Step 2) and press **Enter**:
```cmd
taskkill /F /PID 4312
```

### Step 4: Restart WebGoat
Rerun your launch command:
```bash
java -jar webgoat-2026.4.jar
```

---

## 🍏 Method 2: For Mac / Linux Users

### Step 1: Open the Terminal
Press `Cmd + Space`, type **Terminal**, and press **Enter**.

### Step 2: Force-Kill the Process
Run this command to instantly terminate whatever is blocking port 9090:
```bash
kill -9 \$(lsof -t -i:9090)
```
*(If prompted, type your Mac login password and hit Enter).*

### Step 3: Restart WebGoat
Rerun your launch command:
```bash
java -jar webgoat-2026.4.jar
```

---

## 💡 Alternative Fix: Move WebWolf to a Different Port
If you cannot kill the process on port 9090 (or it keeps coming back), you can tell WebWolf to use port **9095** instead. This allows **WebGoat to stay on its standard port 8080** without changing any laptop settings:

```bash
java -Dfile.encoding=UTF-8 -Dwebwolf.port=9095 -jar webgoat-2026.4.jar
```

### Your New URLs:
* **WebGoat Main Interface:** [http://localhost:8080/WebGoat](http://localhost:8080/WebGoat) *(Case-sensitive)*
* **WebWolf Companion:** [http://localhost:9095/WebWolf](http://localhost:9095/WebWolf)
