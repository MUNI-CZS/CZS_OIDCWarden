# OIDCWarden - CZS Interní Verze

Soft fork z [dani-garcia/vaultwarden](https://github.com/dani-garcia/vaultwarden).
Cílem je poskytnout OIDC kompatibilní řešení pro interní použití v CZS (Centrum zpracování sítí).

Master heslo je stále vyžadováno a není kontrolováno SSO (v závislosti na vašem pohledu to může být funkce ;).

Bitwarden [key connector](https://bitwarden.com/help/about-key-connector) není podporován a kvůli [licenci](https://github.com/bitwarden/key-connector/blob/main/LICENSE.txt) je vysoce nepravděpodobné, že kdy bude:

> 2.1 Commercial Module License. Subject to Your compliance with this Agreement, Bitwarden hereby grants to You a limited, non-exclusive, non-transferable, royalty-free license to use the Commercial Modules for the sole purposes of internal development and internal testing, and only in a non-production environment.

## Rychlý Start s Docker Compose

Pro kompletní nastavení včetně OIDC proxy pro externí poskytovatele jako Masarykova univerzita použijte poskytnutý `docker-compose.yml`:

```bash
# Naklonujte repozitář
git clone <repository-url>
cd CZS_OIDCWarden

# Vytvořte .env soubor s vaší OIDC konfigurací
# Viz sekce Konfigurace níže

# Spusťte všechny služby
docker-compose up -d

# Přístup k OIDCWarden na http://localhost:3000
```

### Nginx OIDC Proxy Nastavení

Při integraci s externími OIDC poskytovateli, kteří mají problémy s discovery endpointem (jako Masarykova univerzita `id.muni.cz`), je zahrnut vlastní Nginx proxy pro:

1. **Interceptování OIDC Discovery**: Poskytování kontrolované odpovědi `/.well-known/openid-configuration`
2. **Proxy OIDC Požadavků**: Přeposílání skutečných OIDC flow požadavků na externí poskytovatele
3. **Řešení Síťových Problémů**: Řešení Docker container networking s hostitelskými službami

**Proč je potřeba pro MUNI OIDC:**
- MUNI discovery endpoint vrací 500 chyby
- OIDCWarden vyžaduje validní OIDC discovery pro fungování
- Proxy poskytuje stabilní discovery endpoint při proxy skutečných OIDC požadavků

**Zahrnuté soubory:**
- `docker-compose.yml` - Kompletní nastavení s OIDCWarden a Nginx proxy
- `nginx.conf` - Nginx konfigurace pro OIDC proxy
- `.env` - Environment proměnné pro OIDC konfiguraci

## Poděkování

Projekt byl umožněn díky podpoře [sponzorů](https://github.com/sponsors/Timshel) a [TU Bergakademie Freiberg](https://tu-freiberg.de/en).

Nebylo by možné bez maintainerů a přispěvatelů zdrojového projektu [dani-garcia/vaultwarden](https://github.com/dani-garcia/vaultwarden).

## Verze

Tagované verze jsou založeny na Bitwarden web client releases, např. `v2024.8.3-1` je první release kompatibilní s web clientem `v2024.8.3`.

Viz [changelog](CHANGELOG.md) pro více detailů.

## Rozdíly s [timshel/vaultwarden](https://github.com/timshel/vaultwarden)

Tento projekt byl vytvořen ve snaze získat více podpory pro udržení [PR](https://github.com/dani-garcia/vaultwarden/pull/3899) přidávající OpenID Connect do Vaultwarden.

Hlavní rozdíly nyní:

- Přejmenovaný projekt / změněné ikony
- Schopnost vydávat `web-vault` nezávisle na [bw_web_builds](https://github.com/dani-garcia/bw_web_builds), [timshel/vaultwarden](https://github.com/timshel/vaultwarden) zůstane synchronizovaný.

V dlouhodobém horizontu se distribuce zaměří na lepší OpenID flow (`web-vault` `override` distribuce).

## Testování

Nové release jsou testovány pomocí Playwright integračních testů. Aktuálně testované flow zahrnují:

- Login flow pomocí Master hesla a/nebo SSO
- 2FA pomocí emailu a TOTP (s/bez SSO)
- Role mapping (přístup k admin konzoli)
- Vytváření organizací a kolekcí
- Pozvánky do organizací pomocí Master hesla a SSO
- Automatické pozvánky do organizací
- Synchronizace rolí členství v organizaci (Owner, admin ...)
- Odvolání členství v organizaci

Cílem bude pokračovat ve zvyšování pokrytí testů, ale doporučuji vždy nasadit specifickou verzi a vždy zálohovat/testovat před nasazením nového release.

## Konfigurace

Viz detaily v [SSO.md](SSO.md).

### Konfigurace OIDC pro Masarykovu univerzitu

Pro integraci s OIDC poskytovatelem Masarykovy univerzity (`id.muni.cz`) použijte následující konfiguraci ve vašem `.env` souboru:

```bash
# Povolit SSO
SSO_ENABLED="true"
SSO_ONLY="true"
SSO_SIGNUPS_MATCH_EMAIL="true"

# MUNI OIDC Client Konfigurace
SSO_CLIENT_ID="váš-client-id"
SSO_CLIENT_SECRET="váš-client-secret"
SSO_AUTHORITY="http://localhost:8082/"
SSO_SCOPES="openid email profile"
SSO_FRONTEND="override"

# OIDC Endpointy (proxované přes Nginx)
SSO_AUTHORIZATION_ENDPOINT="https://id.muni.cz/oidc/authorize"
SSO_TOKEN_ENDPOINT="https://id.muni.cz/oidc/token"
SSO_USERINFO_ENDPOINT="https://id.muni.cz/oidc/userinfo"

# Doména a Redirect URI
DOMAIN="http://localhost:3000"
SSO_REDIRECT_URI="http://localhost:3000/identity/connect/oidc-signin"
```

**Důležité:** Nastavte redirect URI ve vaší MUNI OIDC aplikaci na:
```
http://localhost:3000/identity/connect/oidc-signin
```

## Funkce

Mapování rolí a organizací může být čteno z id tokenu nebo user info endpointu.
Synchronizace se provádí ve výchozím nastavení při přihlášení a volitelně při obnovení tokenu (může být nákladné, protože klient může spamovat endpoint).

- `SSO_SYNC_ON_REFRESH`: Povolit obnovení rolí, orgů a skupin při refresh_token.

### Mapování rolí

Umožňuje mapovat role z Access tokenu na uživatele pro udělení přístupu k `VaultWarden` `admin` konzoli.
Podporuje dvě role: `admin` nebo `user`.

Tato funkce je kontrolována následující konfigurací:

- `SSO_ROLES_ENABLED`: kontroluje, zda se mapování provádí, výchozí je `false`
- `SSO_ROLES_DEFAULT_TO_USER`: neblokovat přihlášení v případě chybějících nebo neplatných rolí, výchozí je `true`.
- `SSO_ROLES_TOKEN_PATH=/resource_access/${SSO_CLIENT_ID}/roles`: cesta pro čtení rolí v id tokenu nebo user info (používá se i pro role členství v organizaci).

### Synchronizace organizací

Umožňuje synchronizovat Organizace, Skupiny a Role uživatelů.

#### Pozvánky do organizací

Organizace musí být nejprve ručně vytvořena.
Bude použít email spojený s Organizací pro odesílání dalších notifikací (admin strana).

Flow pozvánky bude vypadat takto:

- Dekódovat JWT Access token a zkontrolovat, zda je přítomen seznam organizací (výchozí cesta je `/groups`).
- Zkontrolovat, zda existuje odpovídající Organizace a zda uživatel není její součástí.
- pokud jsou povoleny maily, pozvat uživatele do Organizace
  - Uživatel bude muset kliknout na odkaz v emailu, který obdržel
  - Notifikace je odeslána na `email` spojený s Organizací, že nový uživatel je připraven připojit se
  - Admin bude muset validovat uživatele pro dokončení připojení uživatele k org.
- Jinak jen přidat uživatele do Organizace
  - Admin bude muset validovat uživatele pro potvrzení připojení uživatele k org.

Jedním z bonusů pozvánky je, že pokud organizace definuje specifickou password policy, pak se bude aplikovat na nové uživatele, když si nastaví své nové master heslo.
Pokud je uživatel součástí dvou organizací, pak je seřadí pomocí role uživatele (`Owner`, `Admin`, `User` nebo `Manager` pro nyní manager je poslední :() a vrátí password policy první.

Tato funkce je kontrolována následující konfigurací:

- `SSO_SCOPES`: Volitelné scope override, pokud jsou potřeba dodatečné scopes, výchozí je `"email profile"`
- `SSO_ORGANIZATIONS_ENABLED`: kontroluje, zda se mapování provádí, výchozí je `false`
- `SSO_ORGANIZATIONS_TOKEN_PATH`: cesta pro čtení organizací a skupin v id tokenu nebo user info, výchozí je `/groups`

#### Odvolání organizací

Pokud je uživatel odstraněn z provider skupiny, členství bude odvoláno. Uživatel ztratí přístup, ale nebude potřeba admin intervence pro udělení přístupu zpět.

Stav členství je zachován (`invited`, `confirmed`, `accepted`) a bude obnoven, pokud je uživatel znovu přidán do skupiny/organizace.

Tato funkce je kontrolována následující konfigurací:

- `SSO_ORGANIZATIONS_REVOCATION`: kontroluje, zda se provádějí odvolání uživatelů (výchozí `false`).

#### Role člena organizace

Vlastní role mohou být odeslány pro nastavení role člena organizace. Pouze jedna role může být definována pro všechny organizace.
Pokud není přítomna, uživateli bude přiřazena role `User`.

Možné hodnoty zahrnují:

- `OrgNoSync`: Zakázat všechnu synchronizaci organizací pro tohoto uživatele.
- `OrgOwner`: mapovat na roli `Owner`, přidání/odebrání této role má některé požadavky (je 2FA aktivována? je to poslední `Owner` org?).
- `OrgAdmin`: mapovat na roli `Admin`.
- `OrgManager`: mapovat na vlastní roli se schopností spravovat všechny kolekce.
- `OrgUser`: výchozí, `User` může být udělen přístup ke všem nebo žádným kolekcím.

Pokud není poskytnuta žádná role, bude výchozí `User` při pozvánce a nebude provedena žádná změna pro existující roli člena.

Tato funkce je kontrolována následující konfigurací:

- `SSO_ROLES_TOKEN_PATH=/resource_access/${SSO_CLIENT_ID}/roles`: cesta pro čtení rolí v Access tokenu (Funkce je aktivní i když je `SSO_ROLES_ENABLED` zakázáno).
- `SSO_ORGANIZATIONS_ALL_COLLECTIONS`: jsou `User` udělen přístup ke všem kolekcím, výchozí je `true` (bude aplikováno pouze když se změní role členství uživatele).

#### Skupiny organizací

Skupiny budou muset být nejprve vytvořeny. Pak pokud jsou přítomny v tokenu, uživatelé mohou být přidáni/odebráni.

Tato funkce je kontrolována následující konfigurací:

- `ORG_GROUPS_ENABLED`: Musí být aktivováno.
- `SSO_ORGANIZATIONS_GROUPS_ENABLED`: Musí být aktivováno, Je zde pro zabránění starým nastavením náhle zkusit synchronizovat skupiny, bude bráno jako vždy aktivované brzy a odstraněno.

#### Mapování organizací a skupin

Existuje více způsobů, jak spárovat danou provider skupinovou hodnotu s organizací nebo skupinou.

Organizace (pouze při modifikaci) a skupiny umožňují nastavit `ExternalId` pro pomoc s tímto spojením.

V závislosti na formátu provider hodnoty budou použity různé logiky:

- jednoduché `toto`:
  - Spáruje Organizaci s odpovídajícím `name` nebo `ExternalId`
  - Spáruje Skupinu s odpovídajícím `ExternalId`.
- path style: `/org/group` nebo `org/group` spáruje pomocí pouze názvů organizace a skupiny.

Pouze `path` style umožňuje spárovat skupinu pomocí jejího názvu. Jednoduchá hodnota může spárovat více Organizací/Skupin, to vygeneruje chybu a zakáže synchronizaci.
Při spárování skupiny bude uživatel považován za součást nadřazené Organizace, i když není uveden v provider skupinách.

#### Deprecations

- `SSO_ORGANIZATIONS_INVITE`: Bude odstraněno s příštím release. Nahrazeno `SSO_ORGANIZATIONS_ENABLED`.
- `SSO_ORGANIZATIONS_ID_MAPPING` Bude odstraněno s příštím release. Pro nyní pokud je přítomno, stále se používá, provádí se pouze mapování organizací a rolí uživatelů.
- `SSO_ORGANIZATIONS_GROUPS_ENABLED`: Bude odstraněno s příštím release. Umožňuje udržet mapování skupin zakázané (stále závislé na `ORG_GROUPS_ENABLED`).
  False zpočátku pro vynucení opt-in k funkci.

## Řešení problémů

### Problémy s OIDC Discovery

Pokud narazíte na chyby "Failed to discover OpenID provider":

1. **Zkontrolujte Nginx Proxy**: Ujistěte se, že Nginx proxy běží na portu 8082
   ```bash
   curl http://localhost:8082/.well-known/openid-configuration
   ```

2. **Ověřte OIDC Authority**: Ujistěte se, že `SSO_AUTHORITY` směřuje na proxy (`http://localhost:8082/`)

3. **Zkontrolujte Redirect URI**: Ujistěte se, že redirect URI ve vašem OIDC provideru přesně odpovídá:
   ```
   http://localhost:3000/identity/connect/oidc-signin
   ```

### Problémy s Docker Networking

Pokud OIDCWarden nemůže dosáhnout na Nginx proxy:

1. **Použijte Host IP**: Nahraďte `localhost` vaší host IP v `SSO_AUTHORITY`
2. **Zkontrolujte stav kontejnerů**: `docker-compose ps`
3. **Zobrazte logy**: `docker-compose logs -f`

### Běžné chybové zprávy

- **"Server returned invalid response: HTTP status code 500"**: Externí OIDC provider má problémy → Použijte Nginx proxy
- **"Redirect URI does not match"**: Aktualizujte redirect URI v konfiguraci OIDC provideru
- **"Query string failed to match route declaration"**: Normální pro přímý přístup k endpointu bez OIDC parametrů

## Docker

Změňte docker soubory pro balení obou front-endů z [Timshel/oidc_web_vault](https://github.com/Timshel/oidc_web_vault/releases).

Ve výchozím nastavení bude používat release, který pouze zviditelní tlačítko `sso`.

Pokud chcete použít verzi s dodatečnými funkcemi zmíněnými, výchozí přesměrování na `/sso` a oprava pozvánky do organizace.
Musíte předat env proměnnou: `-e SSO_FRONTEND='override'` (viz [start.sh](docker/start.sh)).

Docker images dostupné na:

 - Docker hub [hub.docker.com/r/timshel/oidcwarden](https://hub.docker.com/r/timshel/oidcwarden/tags)
 - Github container registry [ghcr.io/timshel/oidcwarden](https://github.com/Timshel/oidcwarden/pkgs/container/oidcwarden)

### Front-end verze

Ve výchozím nastavení je front-end verze fixována pro zabránění regresím (zkontrolujte [CHANGELOG.md](CHANGELOG.md)).

Při buildování docker image může být přepsána předáním `OIDC_WEB_RELEASE` arg.

Např. pro build s latest: `--build-arg OIDC_WEB_RELEASE="https://github.com/Timshel/oidc_web_vault/releases/latest/download"`

## Pro testování VaultWarden s Keycloak

[Readme](docker/keycloak/README.md)

## DB Migrace

ATM Migrace přidávají dvě tabulky `sso_nonce`, `sso_users` a sloupec `invited_by_email` do `users_organizations`.

### Návrat k výchozímu VW

Návrat k výchozímu VW DB stavu lze snadno provést ručně (Udělejte zálohu :) :

```psql
>BEGIN;
BEGIN
>DELETE FROM __diesel_schema_migrations WHERE version in ('20230910133000', '20230914133000', '20240214170000', '20240226170000', '20240306170000', '20240313170000', '20250514120000');
>DROP TABLE sso_nonce;
>DROP TABLE sso_users;
>ALTER TABLE users_organizations DROP COLUMN invited_by_email;
>DROP INDEX organizations_external_id; -- pouze sqlite
>ALTER TABLE organizations DROP COLUMN external_id;
> COMMIT / ROLLBACK;
```

## Konfigurace

### Zitadel

Pro použití funkce mapování rolí budete potřebovat definovat vlastní mapování pro vrácení jednoduchého seznamu rolí.
Více detailů v Zitadel [dokumentaci](https://zitadel.com/docs/guides/integrate/retrieve-user-roles#customize-roles-using-actions); vlastní úprava bude vypadat nějak takto:

```javascript
function flatRoles(ctx, api) {
  if (ctx.v1.user.grants == undefined || ctx.v1.user.grants.count == 0) {
    return;
  }

  let grants = [];
  ctx.v1.user.grants.grants.forEach(claim => {
    claim.roles.forEach(role => {
        grants.push(role)
    })
  })

  api.v1.claims.setClaim('my:zitadel:grants', grants)
}
```
