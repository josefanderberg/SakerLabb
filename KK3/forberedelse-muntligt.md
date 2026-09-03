# Muntlig kodgranskning KK3 – förberedelse

Samtal fredag 4 september 2026, ca 8 minuter. Du får ha koden och reflektionen framför dig.
Det är inget minnesprov – det är ett resonemangsprov.

---

## 0. Det du behöver kunna utantill (30 sekunder)

| | |
|---|---|
| **Appen** | SakerLabb Support – supportportal i .NET 10 / C#, Blazor med statisk server-rendering + klassiska controllers, SQLite. |
| **Avsnitt A** | `ImportService.cs` rad 50–71 – `Ping(string host)`. Åtgärdad **kommandoinjektion**. Fix: bort med `cmd.exe`, argument via `ArgumentList`. Commit `dbf90cb`. CodeQL-alert #14, `cs/command-line-injection`, Critical → "closed as fixed". |
| **Avsnitt B** | `UserRepository.cs` rad 87–101 – `SetRole` och `Delete`. Kvarstående **SQL-injektion**. ZAP-alert 40018, High. |
| **Din förbättring** | Parametriserade kommandon → principen **secure by default**. Extra lager: least privilege på databasen (**defense in depth**). Kostnad: ca en halvdag. |
| **De tre du fixade i KK2** | XXE (`8c9ff6a`), osäker deserialisering (`8856fee`), kommandoinjektion (`dbf90cb`) – alla tre i `ImportService.cs`, alla Critical i CodeQL. |
| **De två du valde bort i KK2** | Fynd 4 SQL-injektion (High) och fynd 5 CSRF (Medium). Omprövningsdatum 2026-09-08. |
| **Verktygen** | CodeQL (statisk, GitHub code scanning, default setup, C#) – 36 alerts. OWASP ZAP (dynamisk, passiv + aktiv) mot `http://localhost:5080` – 17–19 alerts. |

---

## 1. Steg 1: "Visa din kod och beskriv syfte och användningsområde" (~60 sek)

> "Det här är SakerLabb Support, en supportportal i .NET 10. Kodavsnitt A är ping-diagnostiken:
> en supportperson skriver in ett värdnamn och appen kör ping för att se om servern svarar innan
> ett ärende eskaleras. Den nås på `GET /api/diag?host=...`.
>
> Kodavsnitt B ligger i användaradministrationen. En admin ska kunna ändra roll och radera
> användare, formulären på `/admin` postar till `POST /account/role` och `POST /account/delete`.
> Datan i den tabellen är användarkonton med namn, e-post, **personnummer** och lösenordshashar.
>
> Och det viktiga för riskbilden: **ingen av de här endpointsen ligger bakom inloggning i koden.**
> Det finns inget `[Authorize]`-attribut på vare sig `ApiController` eller `AccountController`, och
> ingen auth-middleware i `Program.cs`. `/admin`-sidan har inte heller någon kontroll – den renderar
> allas personnummer och lösenordshashar till vem som helst som skriver in adressen. Under labben
> kör jag appen lokalt, men i skarp drift vore det här internetnåbara funktioner."

**Varför det svaret är starkt:** du beskriver funktion + användningsområde + datans känslighet +
exponering. Det är tre av de fyra sakerna VG-kriteriet vill höra, redan i steg 1.

---

## 2. Steg 2: "Vilken risk finns här och hur allvarlig är den?"

Läraren pekar antingen på A eller på B. Ha båda klara. **Sätt alltid graden med tre faktorer:
exponering, utnyttjbarhet, konsekvens.** Det är exakt vad uppgiften ber om.

### Om hen pekar på B (mest sannolikt – det är ditt "kvarstående brist"-avsnitt)

> "Det är SQL-injektion. `userId` och `role` konkateneras rakt in i kommandotexten, så indata kan
> ändra vad frågan *gör*, inte bara vilken rad den träffar.
>
> Konkret: `Delete` med `userId = "1 OR 1=1"` blir `DELETE FROM Users WHERE Id = 1 OR 1=1` –
> hela tabellen. `SetRole` med `role = "Admin'--"` klipper bort `WHERE`-satsen och gör alla till
> admin. Microsoft.Data.Sqlite kör dessutom flera satser i samma `CommandText`, så
> `1; DROP TABLE Users;--` går igenom också.
>
> Jag bedömer den som **hög**, och jag väger tre saker:
> **Exponering hög** – endpointen kräver ingen inloggning.
> **Utnyttjbarhet hög** – ZAP hittade den automatiskt via `userId`, det krävs ingen skicklighet.
> **Konsekvens hög** – hela användartabellen med personnummer och lösenordshashar kan läsas ut
> eller raderas. Hasharna är dessutom osaltad MD5, så de är i praktiken klartextlösenord. Det blir en
> personuppgiftsincident med anmälningsplikt till IMY inom 72 timmar."

### Om hen pekar på A

> "Där *fanns* kommandoinjektion – det är fyndet jag åtgärdade. Före fixen var `FileName` `cmd.exe`
> och `Arguments` en sträng: `"/c ping -n 2 " + host`. Skalet tolkar `&`, så `localhost & echo
> INJICERAD_MARKOR_777` körde ett extra kommando. Jag körde den attacken och såg markören i
> svaret. Konsekvensen är fjärrkörning av kod med appens behörigheter – full kontroll över servern.
> Hög exponering, hög utnyttjbarhet, hög konsekvens, alltså hög risk.
>
> Nu finns inget skal. Men jag vill vara ärlig om att avsnittet inte är *helt* rent – det finns tre saker
> kvar som jag ser." *(→ se avsnitt 5 nedan, det är där VG:t sitter)*

---

## 3. Steg 3: "Varför valde du just den lösningen, vilka alternativ fanns?" (~60 sek)

> "Funktionen är nätverksdiagnostik – supportpersonen måste fortfarande kunna pinga, så att
> ta bort funktionen var inget alternativ.
>
> Jag hade tre vägar:
>
> **1. Blocklista farliga tecken och behålla skalet.** Valde bort. Blocklistor missar alltid fall, och
> så länge skalet finns kvar är jag en glömd tecken-variant från att vara sårbar igen. Det behandlar
> symptomet, inte orsaken.
>
> **2. `System.Net.NetworkInformation.Ping` i .NET.** Renast tekniskt – ingen extern process alls,
> ingen parsning av utdata, plattformsoberoende. Jag valde bort den *här och nu* eftersom den
> ändrar returtypen och därmed anropande kod och vyn, och jag ville ha en fix jag kunde verifiera
> isolerat med en ny CodeQL-körning. Den står kvar på min lista som nästa steg.
>
> **3. Ta bort skalet och skicka argumenten var för sig via `ArgumentList`.** Det blev mitt val.
> `FileName = "ping"` startar ping-binären direkt, utan `cmd.exe` emellan, och `ArgumentList`
> escapar varje argument var för sig. Det finns då ingen tolk som kan se `&` som en separator –
> `host` blir ett enda argument som ping försöker slå upp som värdnamn, och specialtecken blir bara
> text. Alltså: **indata kan inte längre bli kod.**
>
> Verifieringen: CodeQL-alert #14 gick från öppen till 'closed as fixed' på main efter commit
> `dbf90cb`, och jag körde attacken före och efter – markören syntes före, inte efter. Beviset kommer
> från en ny verktygskörning, inte från min egen bedömning."

**Nyckelmeningen om hen frågar "vad är skillnaden egentligen?":**
`ArgumentList` går förbi strängparsning helt – .NET bygger argumentvektorn själv och skickar den till
processen. `Arguments` (strängen) måste tolkas av något, och när det något är `cmd.exe` blir varje
metatecken en instruktion.

---

## 4. Steg 4: "Ge minst ett konkret förbättringsförslag kopplat till en namngiven princip" (~45 sek)

> "För SQL-injektionen i avsnitt B: sluta bygga frågor med strängkonkatenering och använd
> parametriserade kommandon – `command.Parameters.AddWithValue("@id", userId)` – eller EF Core.
> Värdet skickas då skilt från SQL:en och kan aldrig tolkas som kod.
>
> Principen är **secure by default**. Poängen är inte bara att den här raden blir säker, utan att med
> ett fråge-API som parametriserar automatiskt blir det säkra alternativet det som gäller när ingen
> tänker efter. Med handbyggda strängar är varje ny metod i repositoryt en ny chans att göra fel.
> Kostnaden är låg till medel: ett tiotal metoder i `UserRepository` och `TicketRepository`, ungefär
> en halvdag.
>
> Som ett extra lager – **defense in depth** – vill jag öppna databasen skrivskyddat på läsvägarna
> (`Mode=ReadOnly` i anslutningssträngen) och stänga filrättigheterna på `sakerlabb.db`. Då blir en
> injektion som ändå slipper igenom mindre allvarlig. Det är samma tanke som least privilege, men
> anpassad till SQLite som inte har databasanvändare att GRANT:a på."

⚠️ **Viktig rättelse mot ditt inlämnade underlag.** Där skrev du "ge databasanvändaren least
privilege: bara läs- och skrivrätt i sina tabeller, ingen rätt att ändra schema". **SQLite har inga
databasanvändare och inga GRANT** – det är en fil. Om läraren tar upp det, säg:

> "Där uttryckte jag mig i termer av SQL Server eller Postgres. I den här appen är databasen en
> SQLite-fil, så det finns inga användare att begränsa. Principen håller, men implementationen blir
> en annan: filrättigheter på `sakerlabb.db`, `Mode=ReadOnly` på anslutningar som bara läser, och
> att appen inte körs som root. Hade det varit SQL Server hade det varit en egen databasanvändare
> utan `ALTER`-rätt."

Att själv rätta det innan hen hittar det är ett plus, inte ett minus.

---

## 5. Där ditt underlag är svagt – ha svaren klara

Det här är sannolikt var läraren gräver. Ärliga, förberedda svar här är precis skillnaden mellan G och VG.

### 5.1 "Är avsnitt A verkligen säkert nu?" ← räkna med den här

Nej, inte helt – och att du ser det själv är starkt. Tre saker kvar:

1. **Argumentinjektion.** `host` valideras inte. `ArgumentList` tar bort *skalet*, men ping gör
   fortfarande sin egen argumentparsning – ett värde som börjar med bindestreck landar som en
   **flagga** i stället för som ett värdnamn. Indata styr alltså fortfarande programmets beteende,
   även om den inte längre kan bli ett eget kommando. Ping-utdatan returneras dessutom rå till
   anroparen, så felutskrifter och användningstext läcker information om servermiljön. Fix:
   validera med `Uri.CheckHostName(host) != UriHostNameType.Unknown` innan anropet – en
   tillåtlista, inte en blocklista.
2. **Timeouten gör inte vad den ser ut att göra.** `ReadToEnd()` blockerar tills processen stänger
   sin utström, och först *därefter* körs `WaitForExit(5000)`. Femsekunderstimeouten bromsar
   alltså ingenting – tråden är redan låst när vi kommer dit. Ett värdnamn som är långsamt att slå
   upp eller inte svarar håller anropet öppet i åtta sekunder eller mer (Windows ping väntar 4000 ms
   per paket, och vi skickar två). Utan inloggning och utan rate limit är det en resursutmattningsväg:
   några hundra samtidiga anrop mot `/api/diag` och appen har inga trådar kvar. Fix: läs asynkront,
   sätt en riktig timeout och `Kill()` processen när den löper ut.
3. **`Process` disposas aldrig.** Ingen `using`, så handtag läcker vid varje anrop.

Plus: **ping-endpointen kräver fortfarande ingen inloggning.** Även utan injektion är en öppen
ping-funktion ett rekognoseringsverktyg – en utomstående kan kartlägga vilka interna adresser som
svarar. Fixen tog bort RCE:n, inte exponeringen.

Och en detalj värd att nämna om hen frågar om portabilitet: `-n` är Windows-flaggan för antal
paket. På Linux betyder `-n` "numerisk utdata" och antal sätts med `-c`. Koden är alltså bunden till
Windows – ännu ett argument för `System.Net.NetworkInformation.Ping`.

### 5.2 "I KK2 skrev du att endpointsen kräver inloggning. I KK3 skriver du att de inte gör det. Vilket gäller?"

**KK3 gäller. Säg det rakt ut:**

> "KK3 är det korrekta. I KK2 angav jag inloggning som kompenserande kontroll för att jag antog att
> admin-funktionerna satt bakom auth. När jag läste koden noggrant inför den här granskningen såg
> jag att det inte stämmer: det finns inget `[Authorize]` någonstans, ingen
> `UseAuthentication`/`UseAuthorization` i `Program.cs`, och `/admin` gör ingen rollkontroll alls.
> Den kompenserande kontrollen höll alltså inte. Det som faktiskt begränsar risken i labbmiljön är
> bara att appen körs lokalt. Det gör att jag skulle flytta upp SQL-injektionen i prioritering – jag
> underskattade dess exponering i KK2."

Att revidera en bedömning när underlaget ändras är omdöme, inte en miss. Det är precis vad
omprövningsdatumet 2026-09-08 finns till för.

### 5.3 "Varför prioriterade du SQL-injektionen som nummer fyra om du bedömer den som hög?"

> "Därför att allvarlighetsgrad och prioritet inte är samma sak. Allvarlighetsgraden är en egenskap
> hos fyndet; prioriteten är mitt beslut om var jag lägger min tid. De tre översta gav angriparen
> **kontroll över servern** – körning av kod. SQL-injektionen ger tillgång till **data**. Med samma
> arbetsinsats stoppar jag alltså mer skada uppåt i listan. Dessutom satt alla tre översta i samma fil,
> `ImportService.cs`, vilket gav en ren commit per åtgärd och spårbar verifiering med en ny
> CodeQL-körning. SQL-injektionen är utspridd över hela `UserRepository` och förtjänar en egen
> genomgång. Med det jag vet nu om att endpointsen är oautentiserade skulle jag dock lyfta den."

### 5.4 "Vad kan en angripare faktiskt göra med SQL-injektion i SQLite?"

> "Mindre än i SQL Server, och det ska man vara ärlig om. Det finns inget `xp_cmdshell` och ingen
> `LOAD_FILE`, så det leder inte till kodkörning. Men: läsa ut hela tabeller via `UNION SELECT`,
> ändra eller radera data, och `ATTACH DATABASE` kan skriva en ny databasfil till disk. Det är alltså
> en dataincident, inte ett serverövertagande – vilket är exakt varför jag rankade den under de tre
> Critical-fynden men ändå som hög allvarlighetsgrad."

### 5.5 "Sista meningen i din reflektion – 'sanera och validera på väg in, neutralisera på väg ut' – vad menar du konkret?"

Den meningen står lös i ditt underlag och är precis den sortens generella rekommendation uppgiften
varnar för. Förankra den direkt i kod:

> "Den syftar på `ApiController.Echo`, rad 61–67. Där skrivs söksträngen `q` rakt in i ett HTML-svar
> med `Response.WriteAsync`, utan kodning – det är reflekterad XSS. Poängen med formuleringen är
> att validering på väg in och kodning på väg ut är två olika kontroller för två olika saker: validering
> avgör om datan är rimlig, kodning avgör hur den tolkas i sin målkontext. Man kan inte ersätta den
> ena med den andra. I `Echo` är rätt åtgärd HTML-kodning vid utskrift, eller ännu hellre att inte
> bygga HTML för hand alls utan låta Blazor rendera – då escapas det som standard, secure by default."

### 5.6 Om hen ber om ett **fail secure**-exempel

Du har ett mönsterexempel i din egen kod. `AuthService.cs` rad 65–77:

```csharp
public bool IsAdmin()
{
    try { var user = Current(); return user is not null && user.Role == "Admin"; }
    catch (Exception exception)
    {
        _logger.LogWarning("Kunde inte avgöra behörighet: {Message}", exception.Message);
        return true;   // ← släpper igenom vid fel
    }
}
```

> "Det här är fail *open*, raka motsatsen till fail secure. Och det går att trigga: `Current()` gör
> `Convert.FromBase64String` och `int.Parse` på cookien, så en trasig cookie kastar undantag – och
> då returnerar behörighetskontrollen `true`. En angripare skickar alltså skräp i `sl_session` och blir
> admin. Fixen är en rad: `return false`. När behörighetstjänsten inte kan svara ska svaret vara nej.
>
> Fast det verkliga problemet är värre än så: `IsAdmin()` anropas inte från något ställe i appen.
> `/admin` har ingen kontroll alls. Så jag skulle inte bara rätta `catch`-satsen utan faktiskt använda
> metoden – och hellre ersätta hela hemmabygget med ASP.NET Cores auth, för sessionscookien är
> bara base64 av `användarnamn|roll|id`, helt osignerad. Vem som helst kan skriva `admin|Admin|1`,
> base64-koda och vara admin. Cookien sätts dessutom med `HttpOnly = false`, `Secure = false` och
> `SameSite = None`."

---

## 6. De fyra principerna – en mening var

| Princip | Vad den betyder | Ditt exempel i den här koden |
|---|---|---|
| **Defense in depth** | Flera lager, inget enskilt skydd bär ensamt | Parametriserade frågor **+** skrivskyddad DB-anslutning **+** filrättigheter på `sakerlabb.db` |
| **Least privilege** | Exakt de rättigheter uppgiften kräver, inte mer | Appen ska inte köras som root; i SQL Server: DB-användare utan `ALTER`. I SQLite: filrättigheter och `Mode=ReadOnly` |
| **Fail secure** | Går något fel ska systemet landa stängt, inte öppet | `AuthService.IsAdmin()` returnerar `true` i sin `catch` – ska vara `false` |
| **Secure by default** | Det säkra alternativet gäller när ingen ställt in något | Parametriserat fråge-API i stället för handbyggda strängar; Blazors automatiska escaping i stället för handbyggd HTML |

---

## 7. Snabba svar på "vad skulle du göra härnäst?"

Prioriterad lista om läraren ber om en åtgärdsplan:

1. **Autentisering och auktorisering på riktigt.** Ingen endpoint är skyddad idag och `/admin`
   läcker personnummer och lösenordshashar till vem som helst. Det slår ut alla andra fixar i
   allvarlighet, eftersom det är förutsättningen som gör de övriga bristerna utnyttjbara. (Broken
   Access Control, OWASP A01.)
2. **Parametrisera alla frågor** i `UserRepository` och `TicketRepository`. En halvdag.
3. **Lösenordshashning.** `CryptoService.HashPassword` är osaltad MD5 – byt till Argon2id eller
   ASP.NET Cores `PasswordHasher`. Och `UserRepository.cs` rad 28 loggar lösenordet i klartext.
4. **CSRF-token** på admin-formulären. `app.UseAntiforgery()` finns i pipelinen, men den
   auto-validerar bara Razor Components-endpoints – klassiska MVC-controllers behöver
   `[AutoValidateAntiforgeryToken]`. Det förklarar varför ZAP:s fynd 5 står kvar.
5. **Stäng av `UseDeveloperExceptionPage()` i produktion** och sluta returnera
   `exception.ToString()` från `/api/report` rad 125 – det läcker stacktrace till användaren.
6. **`/api/config`** returnerar SMTP-lösenord och integrations-API-nyckel i klartext (rad 101–112).
   Ta bort endpointen och flytta hemligheterna ur källkoden.
7. **CORS `AllowAnyOrigin`** i `Program.cs` rad 20–26 – begränsa till kända ursprung.

---

## 8. Sista minuten före samtalet

- Ha **koden uppe** i två flikar: `ImportService.cs` och `UserRepository.cs`. Skrolla till rad 50 och 87.
- Ha **reflektionen utskriven** bredvid dig. Du får titta i den.
- **Säg "jag vet inte" när du inte vet**, och lägg till hur du skulle ta reda på det. Det bedöms
  bättre än en gissning som låter säker.
- **Sätt aldrig en allvarlighetsgrad utan tre motiveringar.** Exponering, utnyttjbarhet, konsekvens.
  Varje gång.
- **Namnge principen** i varje förbättringsförslag. Ett förslag utan namngiven princip räknas inte
  mot VG.
- **Nämn alltid ett bortval.** "Jag övervägde X men valde bort det därför att Y" är ett eget
  VG-kriterium.
