# Automatyczna inicjalizacja pierwszego tenanta w nAxiom

Ten katalog zawiera konfigurację Docker Compose z automatyczną inicjalizacją pierwszego tenanta przy użyciu mechanizmu **FirstTenantInitializer** bez zmian w kodzie aplikacji.

## 🎯 Jak to działa?

1. **Docker Configs** - Pliki konfiguracyjne tenanta są definiowane jako `configs` w `docker-compose.yaml`
2. **Montowanie do toMigrate** - Configs są montowane do folderu `/home/app/toMigrate` w kontenerze `tenant-api`
3. **FirstTenantInitializer** - Przy starcie aplikacji automatycznie:
   - Sprawdza czy tenanci już istnieją
   - Czyta pliki z folderu `toMigrate`
   - Tworzy pierwszego tenanta jako **AKTYWNY** (`IsActive = true`)
   - **Automatycznie szyfruje** connection string przed zapisem do bazy
   - Usuwa pliki po utworzeniu tenanta

## 📁 Struktura plików

```
naxiom-init/
├── docker-compose.yaml    # Główna konfiguracja z configs dla pierwszego tenanta
├── .env.example          # Przykładowy plik zmiennych środowiskowych
└── README.md            # Ten plik
```

## 🚀 Szybki start

### 1. Skopiuj plik `.env.example` do `.env`

```bash
cp .env.example .env
```

### 2. Edytuj `.env` - ustaw swoje wartości

Najważniejsze zmienne:

```bash
# Włącz automatyczną inicjalizację
NAX_AUTO_INIT_FIRST_TENANT=true

# Dane pierwszego tenanta
NAX_FIRST_TENANT_CODE=demo
NAX_FIRST_TENANT_NAME=Demo Tenant
NAX_FIRST_TENANT_DATABASE=nAxiom_Demo

# Baza danych
NAX_MSSQL_HOST=mssql
NAX_MSSQL_USERNAME=sa
NAX_MSSQL_PASSWORD=YourStrongPassword123!
```

### 3. Uruchom środowisko

```bash
# Tylko TenantAdmin (tenant-api + tenant-admin UI)
docker-compose --profile tenantAdmin up -d

# Minimal (backend + frontend bez public-api)
docker-compose --profile minimal up -d

# Full (wszystkie serwisy)
docker-compose --profile full up -d
```

### 4. Sprawdź logi

```bash
# Sprawdź czy tenant został utworzony
docker-compose logs tenant-api | grep -i "first tenant"
```

Powinieneś zobaczyć:
```
First tenant created from ENV. Tenant name: 'Demo Tenant', code: 'demo'
```

## 🔐 Bezpieczeństwo

### Connection String - automatyczne szyfrowanie

- ✅ W configs: **plain text** (tylko w definicji YAML)
- ✅ W bazie danych: **automatycznie zaszyfrowany** przez `ITenantAdminCryptography`
- ✅ Pliki toMigrate: **automatycznie usuwane** po utworzeniu tenanta

```csharp
// FirstTenantInitializer.cs - linia 101
connectionString: _tenantAdminCryptography.Encrypt(connectionString)
```

### Klucz szyfrujący

```csharp
// TenantAdminCryptography.cs - hard-coded w aplikacji
private const string _password = "59@KeMqD10*Q$Ge$A#vUu";
```

## 📋 Wymagania

### Przed uruchomieniem upewnij się, że:

1. **Baza danych SQL Server jest dostępna** na `NAX_MSSQL_HOST`
2. **Baza Tenants istnieje** (`NAX_MSSQL_DATABASE`)
3. **Sieci Docker są utworzone**:
   ```bash
   docker network create naxiom-common-db
   docker network create naxiom-common-npm
   ```

## 🔧 Konfiguracja zmiennych środowiskowych

### Wymagane zmienne

| Zmienna | Opis | Przykład |
|---------|------|----------|
| `NAX_AUTO_INIT_FIRST_TENANT` | Włącza automatyczną inicjalizację | `true` |
| `NAX_FIRST_TENANT_CODE` | Kod tenanta (ID) | `demo` |
| `NAX_FIRST_TENANT_DATABASE` | Nazwa bazy danych tenanta | `nAxiom_Demo` |
| `NAX_MSSQL_HOST` | Adres SQL Server | `mssql` |
| `NAX_MSSQL_USERNAME` | Login do SQL | `sa` |
| `NAX_MSSQL_PASSWORD` | Hasło do SQL | `YourPassword123!` |

### Opcjonalne zmienne

| Zmienna | Opis | Domyślna wartość |
|---------|------|------------------|
| `NAX_FIRST_TENANT_NAME` | Nazwa wyświetlana tenanta | `NAX_FIRST_TENANT_CODE` |
| `NAX_FIRST_TENANT_CULTURE` | Kultura (język) | `pl-PL` |
| `NAX_FIRST_TENANT_ADMIN_EMAIL` | Email administratora | `admin@demo.com` |
| `NAX_SMTP_SERVER` | Serwer SMTP | `smtp.office365.com` |
| `NAX_SMTP_PORT` | Port SMTP | `587` |
| `NAX_SMTP_USERNAME` | Login SMTP | `` |
| `NAX_SMTP_PASSWORD` | Hasło SMTP | `` |

## 🧪 Weryfikacja

### Sprawdź czy tenant został utworzony

```bash
# Logi tenant-api
docker-compose logs tenant-api

# Wejdź do bazy danych
docker exec -it <mssql_container> /opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P 'YourPassword123!'
```

```sql
USE nAxiom_Tenants;
SELECT Id, Code, Name, IsActive, AdminEmail 
FROM Tenants;
```

Powinieneś zobaczyć:
```
Id                                   Code    Name         IsActive  AdminEmail
------------------------------------ ------- ------------ --------- -----------------
<guid>                               demo    Demo Tenant  1         admin@demo.com
```

## 🔄 Re-inicjalizacja

### Jeśli chcesz utworzyć tenanta ponownie:

1. **Usuń istniejącego tenanta** z bazy danych:
   ```sql
   USE nAxiom_Tenants;
   DELETE FROM Tenants WHERE Code = 'demo';
   ```

2. **Restart tenant-api**:
   ```bash
   docker-compose restart tenant-api
   ```

3. **Sprawdź logi**:
   ```bash
   docker-compose logs -f tenant-api
   ```

## ⚠️ Ważne uwagi

1. **Pliki toMigrate są usuwane** - Po utworzeniu tenanta, pliki w folderze `toMigrate` są automatycznie usuwane przez aplikację
2. **Tylko pierwszy tenant** - Mechanizm działa tylko gdy **nie istnieją żadni tenantci** w bazie
3. **Migracje baz danych** - Musisz ręcznie uruchomić migracje dla baz tenanta (auth, api, bpmn)
4. **Connection string szyfrowany** - Plain text w `.env` jest OK, zostanie automatycznie zaszyfrowany
5. **IsActive = true** - Tenant jest tworzony jako aktywny, gotowy do użycia

## 🆚 Różnice vs naxiom-dev

| Aspekt | naxiom-dev | naxiom-init |
|--------|------------|-------------|
| Inicjalizacja tenanta | ❌ Ręczna | ✅ Automatyczna |
| Pliki toMigrate | ❌ Brak | ✅ Docker Configs |
| Zmienne ENV dla tenanta | ❌ Brak | ✅ Pełna konfiguracja |
| Pierwszy start | Wymaga logowania do UI | Tenant gotowy po starcie |

## 📚 Dokumentacja

### Pliki configs montowane do toMigrate:

- `api_appsettings.json` - Konfiguracja API tenanta
- `api_appsettings-protected.json` - Connection string + sensitive data dla API
- `auth_appsettings.json` - Konfiguracja AUTH tenanta
- `auth_appsettings-protected.json` - Connection string + SMTP dla AUTH
- `bpmn_appsettings.json` - Konfiguracja BPMN tenanta
- `bpmn_appsettings-protected.json` - Connection string dla BPMN

### Kod źródłowy automatyzacji:

- `repository/TenantAdmin/TenantAdmin.Infrastructure/Services/FirstTenantInitializer.cs`
- Wywoływany przez `TenantAdmin.Api` przy starcie aplikacji
- Logika: sprawdza folder `toMigrate`, czyta pliki, szyfruje, tworzy tenant, usuwa pliki

## 🐛 Troubleshooting

### Tenant nie został utworzony

```bash
# Sprawdź logi
docker-compose logs tenant-api | grep -i tenant

# Sprawdź czy pliki są zamontowane
docker exec tenant-api ls -la /home/app/toMigrate

# Sprawdź zmienne środowiskowe
docker exec tenant-api env | grep NAX_
```

### Connection string nie działa

- Sprawdź czy baza `NAX_FIRST_TENANT_DATABASE` istnieje
- Sprawdź hasło SQL (nie może zawierać znaków specjalnych wymagających escapowania)
- Sprawdź połączenie sieciowe do SQL Server

### Tenant już istnieje

- Mechanizm działa **tylko raz** - jeśli jakikolwiek tenant już istnieje, nie zostanie utworzony nowy
- Usuń istniejące tenantów ręcznie z bazy lub użyj innej bazy Tenants

## 📞 Wsparcie

W razie problemów sprawdź:
1. Logi: `docker-compose logs tenant-api`
2. Baza danych: Tabela `Tenants` w `nAxiom_Tenants`
3. Pliki toMigrate: `docker exec tenant-api ls -la /home/app/toMigrate`
