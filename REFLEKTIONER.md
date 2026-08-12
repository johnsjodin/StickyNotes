1. SENSITIVE DATA EXPOSURE

Problemet var att lösenordet till en test-användare låg i klartext i appsettings.json, vilket följde med in i git-repot och var öppet och tillgängligt.

Ofta när man använder en testanvändare under utveckligen så har den användaren förhöjda rättigheter, vilket gör att ett läckt lösenord till en sån användare kan ställa till mycket oreda i händerna på fel person. Lösenord brukar även återanvändas, vilket gör att läckta lösenord även jämförs mot andra servrar och tjänster vilket riskerar en ännu större säkerhetsläcka och möjlighet till sabotage.

Man bör förebygga att hemligheter läcks genom att separera hemligt innehåll i en appsettings.json till en .Development.json som läggs till i .gitignore, sedan byter man lösenordet i den nya filen då det gamla lösenordet fortfarande återfinns i gamla commits. Man kan även använda GitHubs Secret Scanning-function + pre-commit hooks vilket automatiskt scannar efter hemligheter och varnar innan de hinner committas.


2. BROKEN ACCESS CONTROL

Här var problemet att en inloggad användare kunde uppdatera och radera andra användares notes. Enligt .md-filen skulle det handla om en GET som var sårbar, men så var alltså icke fallet då den sårbarheten hade fixats i en tidigare commit, men PATCH och DELETE hade lämnats öppna.

En sårbarhet som denna kan ju leda till totalt kaos i t.ex. en app för social media, speciellt om PATCH missbrukas och andra användares poster redigeras till något helt annat än vad de själva hade skrivit. Och extra speciellt om den nya posten innehåller en skadlig kodinjektion av något slag.

Fixen är att lägga till samma validering på alla api-anrop, att man alltså kollar att den inloggade användaren är samma som skrivit posten. Men om man gör det manuellt kan det lätt glömmas, vilket hände i det här fallet. Istället kan man använda auktoriserings-policies som definierar vem som äger vad på ett enda ställe, och sen återanvänds av alla endpoints. Det ska även fångas upp av tester.


3. CROSS SITE SCRIPTING

Den här sårbarheten är den jag nämnde ovan, vilken möjliggör att en person kan injicera skadlig kod som körs på andras datorer när de läser en post som egentligen bara ska innehålla text. För mig var det iframe-koden som funkade först. Men det här påmninner mig om ett speciellt sätt att fuska i vissa spel, t.ex. Stardew Valley, där man döper sin karaktär i spelet till ett kodblock istället för ett vanligt namn, och varje gång det "namnet" körs, alltså nämns av en NPC i en dialogruta, så får man en miljon guldkronor eller nåt. Lite random att ta upp kanske, men det tillhör ju samma typ av exploit.

Det klassiska målet med XSS-injektion är att få tillgång till en persons session och/eller lösenord, och på så sätt kapa deras konto, eftersom koden körs med offrets egna identitet.

Fixen kan vara så enkel som att, som md-filen säger, att se till att browsern skriver ut text som text och inte som html, samt att vissa ramverk inom webutveckling hjälper till att förhindra XSS-injektion som standard.


4. SQL INJECTION

Ännu en injektions-exploit, liknande XSS men angriper databasen istället för browsern. Problemet här är att appen bygger sina queries med ren strängkonkatenering, alltså att användarens sökord klistras rakt in i själva SQL-satsen. Då kan databasen inte skilja på vad som är kod och vad som är data från användaren, så via input-fältet kan man då injicera en helt annan query än den som var tänkt och komma åt sånt man inte är behörig till.

Konsekvenserna är värre än med XSS för det är själva datalagret som träffas, där all data faktiskt ligger. Det handlar inte bara om att tjuvkika på andras notiser, utan man kan potentiellt dra ut hela databasen, typ lösenordstabeller, kreditkortsnummer och annat känsligt, eller till och med ändra och radera data. Det är därför SQL-injektion brukar rankas som en av de allra farligaste sårbarheterna.

Lösningen är parametriserade frågor (prepared statements). Då skickas frågans struktur och användarens data separat till databasen, så datan behandlas alltid som ett värde och aldrig som kod. Då spelar det ingen roll vad man skriver i input-fältet, injicerad SQL slutar helt enkelt funka. En ORM som EF Core fixar det här automatiskt så länge man skriver sina queries i LINQ istället för rå SQL, vilket är precis det som fixen i md-filen gjorde.


5. INSECURE DESERIALIZATION

Problemet här var att deserialiseraren (Json.NET) var inställd på TypeNameHandling.Auto, vilket lät den inkommande JSON-datan själv peka ut vilken klass den skulle bli, via ett specialfält som heter $type. Med andra ord var det angriparen, inte utvecklaren, som fick bestämma vilken kod som kördes. Genom att skicka in $type som pekade på en intern klass fick jag servern att skapa ett objekt av den och köra dess kod, vilket i det här fallet bara skrev ut "Access granted" i konsolen.

När ett objekt skapas körs dess konstruktor och setters, och med rätt befintliga klasser kan en angripare kedja ihop det till att köra godtycklig kod direkt på servern, t.ex. läsa filer, ladda in kod eller ta över maskinen. Men mekanismen är alltså annorlunda än de tidigare sårbarheterna, då det handlar om kodkörning på själva servern och inte bara i en webbläsare eller mot en databas.

Fixen var att sätta TypeNameHandling.None, så att indatan aldrig får välja typ, utan bara de klasser som koden själv deklarerat används. Mer generellt bör man undvika att deserialisera data man inte litar på till öppna typer som object, och hellre använda tydligt definierade dataklasser. Måste man ändå ha typinformation med i datan får man vitlista exakt vilka typer som är tillåtna. Summan av kardemumman är att data aldrig ska få bestämma vilken kod som körs.


6. VULNERABLE AND OUTDATED COMPONENTS

Den här var annorlunda, för felet fanns inte i min egen kod utan i ett beroende. Det handlar om typosquatting, alltså att en angripare lägger upp ett fejkpaket med ett namn som är nästan rättstavat och hoppas att någon skriver fel när de lägger till det. Här var det Swashbukle.AspNetCore istället för det äkta Swashbuckle.AspNetCore (ett c saknades), och det fanns faktiskt två sådana felstavade paket. Fejkpaketet hade smugit in sig i System.Linq-namespacet och kapat den helt vanliga .ToArray()-metoden, så koden kördes utan att jag anropat något skumt alls.

Det farliga är att man kör kod man inte skrivit, med samma rättigheter som resten av appen. Hos oss skrev fejkpaketet bara ut ett meddelande, men ett riktigt fejkpaket skulle lika gärna kunna stjäla data, lägga in en bakdörr eller köra kod på servern. Supply chain-attacker är extra lömska eftersom man litar på sina beroenden mer eller mindre blint, och det är svårt att upptäcka en enda felstavad bokstav i en lång paketlista. Log4Shell och dependency confusion-attackerna mot Apple och Microsoft är exempel på hur illa det kan gå i verkligheten.

Fixen var att byta ut de felstavade paketnamnen mot de äkta, båda två. En läxa jag fick på köpet var att man inte bara ska ta bort ett beroende man är osäker på, för jag råkade radera det ena istället för att rätta det, och då slutade bygget funka eftersom koden faktiskt använde det. För att förebygga det här bör man använda automatisk beroendeskanning som Dependabot, låsa fast versioner och vara noga med att dubbelkolla paketnamn och utgivare innan man lägger till något.