# **Juice Shop AppSec Pipeline Lab**

A hands-on **application security learning project** using:  
- [**OWASP Juice Shop**](https://owasp.org/www-project-juice-shop/) *(intentionally vulnerable web app)*  
- [**OWASP ZAP**](https://www.zaproxy.org/) *(automated scanning tool)*  
- [**Burp Suite Community**](https://portswigger.net/burp/communitydownload) *(manual testing tool)*  

The goal is to practice finding vulnerabilities, running automated scans, and documenting results.

---

## **Setup**

### 1. **Prerequisites**
- [Docker & Docker Compose](https://docs.docker.com/get-docker/)  
- (Optional) [Burp Suite Community](https://portswigger.net/burp/communitydownload)  

### 2. **Run Juice Shop + ZAP**
From the repo root:

```bash
docker-compose up zap-baseline
```

This will:  
- Start **Juice Shop** on [http://localhost:3000](http://localhost:3000)  
- Run a **ZAP baseline scan**  
- Save results to `./scans/zap/zap_report.html`  

If you just want to play with Juice Shop manually:  
```bash
docker-compose up juice-shop
```

---

## **Tools**

- **Burp Suite Community** → Manual web testing *(intercept requests, modify inputs, test XSS, SQLi, IDOR, etc.)*  
- **OWASP ZAP** → Automated scanning *(baseline & active scans)*  
- **Docker Compose** → Spins up Juice Shop & scanning environment  

---

## **Learning Goals**
- Understand how vulnerabilities like **SQL Injection, XSS, and IDOR** work  
- Practice using **Burp Suite** & **ZAP** for both manual and automated testing  
- Document findings with **screenshots and reports** for later review  

---

## **Screenshots**
Screenshots of Burp Suite, ZAP alerts, and exploit attempts are stored under:  
- `scans/burpsuite/` & `scans/zap/`*(raw screenshots)*  
- `docs/` *(embedded into walkthroughs)*  

---

## **Disclaimer**
This repo is for **educational purposes only**.  
Do not use these techniques against systems you don’t own or have permission to test.
