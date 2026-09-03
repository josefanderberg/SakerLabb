# Granskningsunderlag – Kunskapskontroll 3

**Namn:** Josef Anderberg
**Klass:** SYNE25LIN
**Kodavsnitt A (säkerhetsfunktion):** `SakerLabb.Web/Services/ImportService.cs`, rad 50–71
**Kodavsnitt B (kvarstående brist):** `SakerLabb.Web/Data/UserRepository.cs`, rad 87–101
**Repo:** https://github.com/josefanderberg/SakerLabb (branch `sakerhetsanalys`, mergad till `main`)

> Detta är en avskrift av den PDF som lämnades in i Learnpoint 2026-09-02.
> Kodavsnitten är verifierade mot `main` — rad- och radnummer stämmer exakt.

---

## Kodavsnitt A – säkerhetsfunktion (åtgärdad kommandoinjektion)

```csharp
// SakerLabb.Web/Services/ImportService.cs, rad 50–71
public string Ping(string host)
{
    var process = new Process
    {
        StartInfo = new ProcessStartInfo
        {
            FileName = "ping",
            RedirectStandardOutput = true,
            UseShellExecute = false,
            CreateNoWindow = true
        }
    };

    process.StartInfo.ArgumentList.Add("-n");
    process.StartInfo.ArgumentList.Add("2");
    process.StartInfo.ArgumentList.Add(host);

    process.Start();
    var output = process.StandardOutput.ReadToEnd();
    process.WaitForExit(5000);
    return output;
}
```

## Kodavsnitt B – kvarstående brist (SQL-injektion)

```csharp
// SakerLabb.Web/Data/UserRepository.cs, rad 87–101
public void SetRole(string userId, string role)
{
    using var connection = _db.Open();
    var command = connection.CreateCommand();
    command.CommandText = "UPDATE Users SET Role = '" + role + "' WHERE Id = " + userId;
    command.ExecuteNonQuery();
}

public void Delete(string userId)
{
    using var connection = _db.Open();
    var command = connection.CreateCommand();
    command.CommandText = "DELETE FROM Users WHERE Id = " + userId;
    command.ExecuteNonQuery();
}
```

---

## Skriftlig reflektion

### 1. Kodens syfte och användningsområde

Båda avsnitten kommer från SakerLabb Support, en supportportal i .NET 10 och C#. Kodavsnitt A är
appens ping-diagnostik: en supportperson skriver in ett värdnamn och appen kör ping för att se om
servern svarar, innan ett ärende eskaleras. Kodavsnitt B ligger i användaradministrationen, där en
admin kan ändra roller och radera användare. Datan som hanteras är användarkonton med namn,
e-post, personnummer och lösenordshashar. Ingen av endpointsen ligger bakom inloggning i koden –
de nås via vanliga URL:er (till exempel `/api/diag` och `/account/delete`). Under laborationen kör jag
appen lokalt, men i skarp drift vore det här funktioner som kan nås via internet.

### 2. Identifierade risker med bedömd allvarlighetsgrad

**Risk 1, kommandoinjektion (avsnitt A, före min fix).** Före fixen byggde koden ihop ping-kommandot
som en textsträng och körde det genom `cmd.exe`. Ett skal tolkar specialtecken, så ett värdnamn som
`localhost & whoami` fick skalet att köra ett extra kommando. Då drabbas hela servern och all data på
den. Exponeringen är hög (ingen inloggning), utnyttjbarheten hög (det räcker med ett `&`) och
konsekvensen hög (körning av godtyckliga kommandon). Därför bedömer jag den totala risken som hög.

**Risk 2, SQL-injektion (avsnitt B, finns kvar).** `userId` och `role` klistras rakt in i SQL-frågan. En
angripare kan ändra vad frågan gör, inte bara vilken rad den träffar, och kan läsa ut eller radera hela
användartabellen inklusive personnummer och lösenordshashar. Det gör att alla användare kan
drabbas, och det blir en personuppgiftsincident med anmälningsplikt till IMY. Exponeringen är hög
(ingen inloggning), utnyttjbarheten hög (ZAP hittade den automatiskt via `userId`) och konsekvensen
hög. Jag bedömer därför den totala risken som hög.

### 3. Vald åtgärd med motivering

För kommandoinjektionen (avsnitt A) tog jag bort skalet helt: `FileName` är nu `"ping"` i stället för
`"cmd.exe"`, och varje argument skickas var för sig via `ArgumentList` i stället för att limmas ihop till
en sträng. Då finns inget skal som tolkar `&` eller `|`, och `host` blir ett enda argument som ping
försöker slå upp som värdnamn – specialtecken blir bara text. Jag övervägde två alternativ. Att
blocklista farliga tecken men behålla skalet valde jag bort, eftersom blocklistor alltid missar fall och är
lätta att kringgå. Att använda .NET:s egna `System.Net.NetworkInformation.Ping` hade varit ännu
renare, eftersom ingen extern process startas alls, men `ArgumentList` var en minimal ändring som
löste själva injektionen och gick att verifiera direkt: CodeQL-larmet gick från öppet till "closed as
fixed" efter fixen. Åtgärden passar funktionen – vi behöver fortfarande kunna pinga – men gör det på
ett sätt där indata aldrig blir kod.

### 4. Förbättringsförslag kopplat till en säkerhetsprincip

För SQL-injektionen (avsnitt B) är förbättringen att sluta bygga frågor med strängkonkatenering och i
stället använda parametriserade kommandon (`Parameters.AddWithValue`) eller EF Core, så att värdet
skickas skilt från SQL:en och aldrig kan tolkas som kod. Principen är **secure by default**: med ett
fråge-API som parametriserar automatiskt blir det säkra alternativet det som gäller utan att någon
behöver tänka efter, till skillnad från handbyggda strängar där varje ny fråga är en ny chans att göra
fel. Kostnaden är låg till medel – det handlar om att skriva om en handfull metoder i `UserRepository`,
ungefär en halvdag. Som ett extra lager (**defense in depth**) skulle jag ge databasanvändaren
**least privilege**: bara läs- och skrivrätt i sina tabeller, ingen rätt att ändra schema. Då blir en
injektion som ändå slipper igenom mindre allvarlig. Och slutligen: sanera och validera data på väg in,
och neutralisera (koda) data på väg ut.
