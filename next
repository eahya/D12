# شغّل هذا السكربت من داخل: C:\Users\Yehya\OneDrive\Desktop\DATA1\YH-BUSINESS-INTELLIGENCE
$root = Get-Location

$dirs = @(
    "sql",
    "backend/app/api",
    "backend/app/core",
    "backend/app/models",
    "backend/app/schemas",
    "backend/app/services",
    "backend/app/ai",
    "backend/app/etl",
    "frontend/electron",
    "frontend/src/components",
    "frontend/src/services"
)

foreach ($d in $dirs) {
    New-Item -ItemType Directory -Force -Path (Join-Path $root $d) | Out-Null
}

function Write-ProjectFile([string]$relPath, [string]$content) {
    $fullPath = Join-Path $root $relPath
    $parent = Split-Path -Parent $fullPath
    if (-not (Test-Path $parent)) {
        New-Item -ItemType Directory -Force -Path $parent | Out-Null
    }
    [System.IO.File]::WriteAllText($fullPath, $content, (New-Object System.Text.UTF8Encoding($false)))
}

# 1. الجذر: .env.example و docker-compose.yml
Write-ProjectFile ".env.example" @'
APP_NAME=YH BUSINESS INTELLIGENCE
BACKEND_PORT=8000
FRONTEND_PORT=80

SQL_SERVER=
SQL_PORT=1433
SQL_DATABASE=YH_BusinessIntelligence
SQL_DRIVER=ODBC Driver 17 for SQL Server
SQL_TRUSTED_CONNECTION=false
SQL_USERNAME=
SQL_PASSWORD=

JWT_SECRET_KEY=
JWT_ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=30
REFRESH_TOKEN_EXPIRE_DAYS=7
'@

Write-ProjectFile "docker-compose.yml" @'
services:
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: yh_bi_backend
    restart: unless-stopped
    env_file:
      - .env
    ports:
      - "${BACKEND_PORT:-8000}:8000"
    networks:
      - yh_bi_network
    healthcheck:
      test: ["CMD-SHELL", "python -c \"import urllib.request; urllib.request.urlopen('http://localhost:8000/health')\""]
      interval: 15s
      timeout: 5s
      retries: 5

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    container_name: yh_bi_frontend
    restart: unless-stopped
    depends_on:
      backend:
        condition: service_healthy
    ports:
      - "${FRONTEND_PORT:-80}:80"
    networks:
      - yh_bi_network

networks:
  yh_bi_network:
    driver: bridge
'@

# 2. سكربتات SQL Server
Write-ProjectFile "sql/01_schema_and_operations.sql" @'
USE master;
GO
IF DB_ID(N'YH_BusinessIntelligence') IS NULL
BEGIN
    CREATE DATABASE YH_BusinessIntelligence;
END
GO
USE YH_BusinessIntelligence;
GO
IF NOT EXISTS (SELECT 1 FROM sys.schemas WHERE name = N'core')       EXEC('CREATE SCHEMA core');
IF NOT EXISTS (SELECT 1 FROM sys.schemas WHERE name = N'security')   EXEC('CREATE SCHEMA security');
IF NOT EXISTS (SELECT 1 FROM sys.schemas WHERE name = N'crm')        EXEC('CREATE SCHEMA crm');
IF NOT EXISTS (SELECT 1 FROM sys.schemas WHERE name = N'sales')      EXEC('CREATE SCHEMA sales');
IF NOT EXISTS (SELECT 1 FROM sys.schemas WHERE name = N'purchasing') EXEC('CREATE SCHEMA purchasing');
IF NOT EXISTS (SELECT 1 FROM sys.schemas WHERE name = N'inventory')  EXEC('CREATE SCHEMA inventory');
IF NOT EXISTS (SELECT 1 FROM sys.schemas WHERE name = N'finance')    EXEC('CREATE SCHEMA finance');
IF NOT EXISTS (SELECT 1 FROM sys.schemas WHERE name = N'hr')         EXEC('CREATE SCHEMA hr');
IF NOT EXISTS (SELECT 1 FROM sys.schemas WHERE name = N'analytics')  EXEC('CREATE SCHEMA analytics');
IF NOT EXISTS (SELECT 1 FROM sys.schemas WHERE name = N'audit')      EXEC('CREATE SCHEMA audit');
GO
CREATE TABLE core.Currencies (
    CurrencyCode VARCHAR(3) NOT NULL PRIMARY KEY,
    CurrencyNameAr NVARCHAR(100) NOT NULL,
    CurrencyNameEn NVARCHAR(100) NOT NULL,
    Symbol NVARCHAR(15) NOT NULL,
    DecimalPlaces TINYINT NOT NULL DEFAULT 2,
    IsActive BIT NOT NULL DEFAULT 1
);
INSERT INTO core.Currencies (CurrencyCode, CurrencyNameAr, CurrencyNameEn, Symbol, DecimalPlaces)
VALUES
    ('EGP', N'جنيه مصري', N'Egyptian Pound', N'ج.م', 2),
    ('USD', N'دولار أمريكي', N'US Dollar', N'$', 2),
    ('SAR', N'ريال سعودي', N'Saudi Riyal', N'ر.س', 2),
    ('EUR', N'يورو', N'Euro', N'€', 2),
    ('GBP', N'جنيه إسترليني', N'British Pound', N'£', 2);
GO
CREATE TABLE core.Companies (
    CompanyID INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    CompanyCode NVARCHAR(30) NOT NULL UNIQUE,
    CompanyNameAr NVARCHAR(200) NOT NULL,
    CompanyNameEn NVARCHAR(200) NOT NULL,
    FunctionalCurrencyCode VARCHAR(3) NOT NULL CONSTRAINT FK_Companies_Currencies FOREIGN KEY REFERENCES core.Currencies(CurrencyCode),
    TaxNumber NVARCHAR(50) NULL,
    CommercialRegNo NVARCHAR(50) NULL,
    IsActive BIT NOT NULL DEFAULT 1,
    CreatedAt DATETIME2(3) NOT NULL DEFAULT SYSUTCDATETIME()
);
CREATE TABLE core.Branches (
    BranchID INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    CompanyID INT NOT NULL CONSTRAINT FK_Branches_Companies FOREIGN KEY REFERENCES core.Companies(CompanyID),
    BranchCode NVARCHAR(30) NOT NULL,
    BranchNameAr NVARCHAR(150) NOT NULL,
    BranchNameEn NVARCHAR(150) NOT NULL,
    IsMainBranch BIT NOT NULL DEFAULT 0,
    IsActive BIT NOT NULL DEFAULT 1,
    CreatedAt DATETIME2(3) NOT NULL DEFAULT SYSUTCDATETIME(),
    CONSTRAINT UQ_Branches_Company_Code UNIQUE (CompanyID, BranchCode),
    CONSTRAINT UQ_Branches_ID_Company UNIQUE (BranchID, CompanyID)
);
CREATE TABLE security.Users (
    UserID INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    CompanyID INT NOT NULL CONSTRAINT FK_Users_Companies FOREIGN KEY REFERENCES core.Companies(CompanyID),
    Username NVARCHAR(80) NOT NULL UNIQUE,
    Email NVARCHAR(150) NOT NULL UNIQUE,
    PasswordHash NVARCHAR(512) NOT NULL,
    FullName NVARCHAR(150) NOT NULL,
    IsActive BIT NOT NULL DEFAULT 1,
    LastLoginAt DATETIME2(3) NULL,
    CreatedAt DATETIME2(3) NOT NULL DEFAULT SYSUTCDATETIME(),
    CONSTRAINT UQ_Users_ID_Company UNIQUE (UserID, CompanyID)
);
GO
'@

# 3. ملفات Backend
Write-ProjectFile "backend/requirements.txt" @'
fastapi>=0.115.0
uvicorn>=0.30.0
sqlalchemy>=2.0.35
pyodbc>=5.1.0
pydantic>=2.9.0
pydantic-settings>=2.5.0
pandas>=2.2.0
numpy>=1.26.0
scikit-learn>=1.5.0
openpyxl>=3.1.5
reportlab>=4.2.0
'@

Write-ProjectFile "backend/Dockerfile" @'
FROM python:3.11-slim
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1 DEBIAN_FRONTEND=noninteractive
WORKDIR /app
RUN apt-get update && apt-get install -y --no-install-recommends curl gnupg2 unixodbc unixodbc-dev build-essential \
    && curl -fsSL https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor -o /usr/share/keyrings/microsoft-prod.gpg \
    && echo "deb [arch=amd64,arm64,armhf signed-by=/usr/share/keyrings/microsoft-prod.gpg] https://packages.microsoft.com/debian/12/prod bookworm main" > /etc/apt/sources.list.d/mssql-release.list \
    && apt-get update && ACCEPT_EULA=Y apt-get install -y --no-install-recommends msodbcsql17 \
    && rm -rf /var/lib/apt/lists/*
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
'@

Write-ProjectFile "backend/app/core/config.py" @'
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    APP_NAME: str = "YH BUSINESS INTELLIGENCE"
    SQL_SERVER: str = "localhost"
    SQL_PORT: int = 1433
    SQL_DATABASE: str = "YH_BusinessIntelligence"
    SQL_USERNAME: str = ""
    SQL_PASSWORD: str = ""
    SQL_DRIVER: str = "ODBC Driver 17 for SQL Server"
    SQL_TRUSTED_CONNECTION: bool = False

    model_config = SettingsConfigDict(env_file=".env", extra="ignore")

    @property
    def sqlalchemy_database_url(self) -> str:
        driver = self.SQL_DRIVER.replace(" ", "+")
        if self.SQL_TRUSTED_CONNECTION:
            return (
                f"mssql+pyodbc://@{self.SQL_SERVER}:{self.SQL_PORT}/{self.SQL_DATABASE}"
                f"?driver={driver}&Trusted_Connection=yes&TrustServerCertificate=yes"
            )
        return (
            f"mssql+pyodbc://{self.SQL_USERNAME}:{self.SQL_PASSWORD}"
            f"@{self.SQL_SERVER}:{self.SQL_PORT}/{self.SQL_DATABASE}"
            f"?driver={driver}&TrustServerCertificate=yes"
        )


settings = Settings()
'@

Write-ProjectFile "backend/app/core/database.py" @'
from typing import Generator
from sqlalchemy import create_engine
from sqlalchemy.orm import declarative_base, sessionmaker, Session
from app.core.config import settings

engine = create_engine(
    settings.sqlalchemy_database_url,
    pool_pre_ping=True,
    pool_size=10,
    max_overflow=20,
    fast_executemany=True,
)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()


def get_db() -> Generator[Session, None, None]:
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
'@

Write-ProjectFile "backend/app/core/security_config.py" @'
import base64
import hashlib
import hmac
import json
import secrets
from datetime import datetime, timedelta, timezone
from typing import Any, Dict, Optional
from fastapi import HTTPException, status
from pydantic_settings import BaseSettings, SettingsConfigDict


class AuthSettings(BaseSettings):
    JWT_SECRET_KEY: str = ""
    JWT_ALGORITHM: str = "HS256"
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 30
    REFRESH_TOKEN_EXPIRE_DAYS: int = 7
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")


auth_settings = AuthSettings()


def _get_signing_key() -> str:
    if not auth_settings.JWT_SECRET_KEY or len(auth_settings.JWT_SECRET_KEY.strip()) < 16:
        raise HTTPException(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            detail="لم يتم ضبط JWT_SECRET_KEY في متغيرات البيئة بطول 16 حرفًا على الأقل.",
        )
    return auth_settings.JWT_SECRET_KEY.strip()


def hash_password(plain_password: str) -> str:
    if not plain_password or len(plain_password) < 8:
        raise ValueError("كلمة المرور يجب أن تتكون من 8 أحرف على الأقل.")
    salt = secrets.token_bytes(16)
    iterations = 200_000
    dk = hashlib.pbkdf2_hmac("sha256", plain_password.encode("utf-8"), salt, iterations)
    return f"pbkdf2_sha256${iterations}${base64.urlsafe_b64encode(salt).decode()}${base64.urlsafe_b64encode(dk).decode()}"


def verify_password(plain_password: str, stored_hash: str) -> bool:
    try:
        scheme, iter_str, salt_b64, digest_b64 = stored_hash.split("$", 3)
        if scheme != "pbkdf2_sha256":
            return False
        iterations = int(iter_str)
        salt = base64.urlsafe_b64decode(salt_b64.encode())
        expected_digest = base64.urlsafe_b64decode(digest_b64.encode())
        computed_digest = hashlib.pbkdf2_hmac("sha256", plain_password.encode("utf-8"), salt, iterations)
        return hmac.compare_digest(computed_digest, expected_digest)
    except Exception:
        return False


def _b64url_encode(raw: bytes) -> str:
    return base64.urlsafe_b64encode(raw).rstrip(b"=").decode("ascii")


def _b64url_decode(segment: str) -> bytes:
    padding = "=" * (-len(segment) % 4)
    return base64.urlsafe_b64decode((segment + padding).encode("ascii"))


def create_access_token(payload: Dict[str, Any], expires_minutes: Optional[int] = None) -> str:
    secret = _get_signing_key()
    ttl = expires_minutes if expires_minutes is not None else auth_settings.ACCESS_TOKEN_EXPIRE_MINUTES
    now_dt = datetime.now(timezone.utc)
    exp_dt = now_dt + timedelta(minutes=ttl)
    header = {"alg": auth_settings.JWT_ALGORITHM, "typ": "JWT"}
    body = {**payload, "iat": int(now_dt.timestamp()), "exp": int(exp_dt.timestamp()), "jti": secrets.token_hex(12)}
    h_seg = _b64url_encode(json.dumps(header, separators=(",", ":")).encode("utf-8"))
    p_seg = _b64url_encode(json.dumps(body, separators=(",", ":")).encode("utf-8"))
    sig = hmac.new(secret.encode("utf-8"), f"{h_seg}.{p_seg}".encode("ascii"), hashlib.sha256).digest()
    return f"{h_seg}.{p_seg}.{_b64url_encode(sig)}"


def decode_access_token(token: str) -> Dict[str, Any]:
    secret = _get_signing_key()
    try:
        h_seg, p_seg, sig_seg = token.split(".")
        expected_sig = hmac.new(secret.encode("utf-8"), f"{h_seg}.{p_seg}".encode("ascii"), hashlib.sha256).digest()
        if not hmac.compare_digest(expected_sig, _b64url_decode(sig_seg)):
            raise ValueError("Invalid signature")
        payload = json.loads(_b64url_decode(p_seg).decode("utf-8"))
        if int(datetime.now(timezone.utc).timestamp()) >= int(payload["exp"]):
            raise ValueError("Expired")
        return payload
    except Exception:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="رمز الوصول غير صالح أو منتهي.")
'@

Write-ProjectFile "backend/app/ai/offline_intelligence_engine.py" @'
from __future__ import annotations
from typing import Any, Dict, List
import numpy as np
import pandas as pd
from sklearn.linear_model import Ridge


class OfflineIntelligenceEngine:
    def __init__(self, sales_df: pd.DataFrame, expenses_df: pd.DataFrame, currency_code: str = "EGP") -> None:
        self.currency_code = currency_code
        self.sales_df = self._prep(sales_df)
        self.expenses_df = self._prep(expenses_df)

    @staticmethod
    def _prep(df: pd.DataFrame) -> pd.DataFrame:
        if df is None or df.empty:
            return pd.DataFrame(columns=["date", "amount"])
        out = df.copy()
        out["date"] = pd.to_datetime(out["date"], errors="coerce")
        out["amount"] = pd.to_numeric(out["amount"], errors="coerce")
        return out.dropna(subset=["date", "amount"]).sort_values("date").reset_index(drop=True)

    def forecast_revenue(self, months_ahead: int = 3) -> Dict[str, Any]:
        if self.sales_df.empty:
            return {"status": "INSUFFICIENT_DATA", "currency": self.currency_code, "forecasts": []}
        m = self.sales_df.assign(period=self.sales_df["date"].dt.to_period("M")).groupby("period", as_index=False)["amount"].sum()
        m = m[m["amount"] > 0].reset_index(drop=True)
        if len(m) < 3:
            return {"status": "INSUFFICIENT_DATA", "currency": self.currency_code, "forecasts": []}
        x = np.arange(len(m), dtype=float).reshape(-1, 1)
        y = m["amount"].to_numpy(dtype=float)
        model = Ridge(alpha=1.0).fit(x, y)
        xf = np.arange(len(m), len(m) + months_ahead, dtype=float).reshape(-1, 1)
        preds = np.maximum(0.0, model.predict(xf))
        last_p = m["period"].iloc[-1]
        return {
            "status": "OK",
            "model": "RidgeRegression",
            "currency": self.currency_code,
            "forecasts": [{"period": str(last_p + i), "predicted_revenue": round(float(v), 2), "currency": self.currency_code} for i, v in enumerate(preds, 1)],
        }
'@

Write-ProjectFile "backend/app/main.py" @'
from fastapi import FastAPI
from app.core.config import settings

app = FastAPI(title=settings.APP_NAME, version="1.0.0")


@app.get("/health")
def health():
    return {"status": "ok", "system": settings.APP_NAME}
'@

# 4. ملفات Frontend و Electron
Write-ProjectFile "frontend/package.json" @'
{
  "name": "yh-business-intelligence-desktop",
  "version": "1.0.0",
  "main": "electron/main.cjs",
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "electron:start": "electron .",
    "dist:win": "npm run build && electron-builder --win nsis --x64"
  },
  "dependencies": {
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "recharts": "^2.12.7"
  },
  "devDependencies": {
    "@types/react": "^18.3.12",
    "@types/react-dom": "^18.3.12",
    "electron": "^32.2.0",
    "electron-builder": "^25.1.8",
    "typescript": "^5.6.3",
    "vite": "^5.4.10"
  },
  "build": {
    "appId": "com.yh.businessintelligence.desktop",
    "productName": "YH BUSINESS INTELLIGENCE",
    "directories": { "output": "release" },
    "files": ["dist/**/*", "electron/**/*"],
    "win": { "target": [{ "target": "nsis", "arch": ["x64"] }] },
    "nsis": {
      "oneClick": false,
      "allowToChangeInstallationDirectory": true,
      "createDesktopShortcut": true,
      "createStartMenuShortcut": true,
      "shortcutName": "YH BUSINESS INTELLIGENCE"
    }
  }
}
'@

Write-ProjectFile "frontend/Dockerfile" @'
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:1.27-alpine
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
'@

Write-ProjectFile "frontend/nginx.conf" @'
server {
    listen 80;
    server_name _;
    root /usr/share/nginx/html;
    index index.html;

    location /api/ {
        proxy_pass http://backend:8000/api/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location / {
        try_files $uri $uri/ /index.html;
    }
}
'@

Write-ProjectFile "frontend/electron/main.cjs" @'
const { app, BrowserWindow, ipcMain } = require("electron");
const fs = require("fs");
const path = require("path");

const CONFIG_FILE = "sql-server-connection.json";

function getConfigPath() {
  return path.join(app.getPath("userData"), CONFIG_FILE);
}

function readConfig() {
  const p = getConfigPath();
  if (!fs.existsSync(p)) return null;
  try {
    return JSON.parse(fs.readFileSync(p, "utf-8"));
  } catch {
    return null;
  }
}

function writeConfig(cfg) {
  const p = getConfigPath();
  const clean = {
    backendUrl: String(cfg.backendUrl || "http://127.0.0.1:8000").trim(),
    sqlServer: String(cfg.sqlServer || "").trim(),
    sqlPort: Number(cfg.sqlPort || 1433),
    sqlDatabase: String(cfg.sqlDatabase || "YH_BusinessIntelligence").trim(),
    sqlDriver: String(cfg.sqlDriver || "ODBC Driver 17 for SQL Server").trim(),
    authMode: cfg.authMode === "SQL_AUTH" ? "SQL_AUTH" : "WINDOWS_AUTH",
    sqlUsername: cfg.authMode === "SQL_AUTH" ? String(cfg.sqlUsername || "").trim() : "",
    sqlPassword: cfg.authMode === "SQL_AUTH" ? String(cfg.sqlPassword || "") : "",
    configuredAt: new Date().toISOString(),
  };
  fs.writeFileSync(p, JSON.stringify(clean, null, 2), { encoding: "utf-8", mode: 0o600 });
  return clean;
}

function createWindow() {
  const win = new BrowserWindow({
    width: 1366,
    height: 860,
    minWidth: 1024,
    minHeight: 700,
    backgroundColor: "#121212",
    title: "YH BUSINESS INTELLIGENCE",
    webPreferences: {
      preload: path.join(__dirname, "preload.cjs"),
      contextIsolation: true,
      nodeIntegration: false,
    },
  });
  if (process.env.VITE_DEV_SERVER_URL) {
    win.loadURL(process.env.VITE_DEV_SERVER_URL);
  } else {
    win.loadFile(path.join(__dirname, "../dist/index.html"));
  }
}

ipcMain.handle("db-config:get", () => readConfig());
ipcMain.handle("db-config:save", (_e, cfg) => writeConfig(cfg));
ipcMain.handle("db-config:reset", () => {
  const p = getConfigPath();
  if (fs.existsSync(p)) fs.unlinkSync(p);
  return true;
});

app.whenReady().then(() => {
  createWindow();
  app.on("activate", () => {
    if (BrowserWindow.getAllWindows().length === 0) createWindow();
  });
});

app.on("window-all-closed", () => {
  if (process.platform !== "darwin") app.quit();
});
'@

Write-ProjectFile "frontend/electron/preload.cjs" @'
const { contextBridge, ipcRenderer } = require("electron");

contextBridge.exposeInMainWorld("yhDesktop", {
  getConnectionConfig: () => ipcRenderer.invoke("db-config:get"),
  saveConnectionConfig: (cfg) => ipcRenderer.invoke("db-config:save", cfg),
  resetConnectionConfig: () => ipcRenderer.invoke("db-config:reset"),
});
'@

Write-Host "تم إنشاء جميع الملفات بنجاح داخل: $root"
