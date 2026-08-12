1. SENSITIVE DATA EXPOSURE

Problemet var att lösenordet till en test-användare låg i klartext i appsettings.json, vilket följde med in i git-repot och var öppet och tillgängligt.

Ofta när man använder en testanvändare under utveckligen så har den användaren förhöjda rättigheter, vilket gör att ett läckt lösenord till en sån användare kan ställa till mycket oreda i händerna på fel person. Lösenord brukar även återanvändas, vilket gör att läckta lösenord även jämförs mot andra servrar och tjänster vilket riskerar en ännu större säkerhetsläcka och möjlighet till sabotage.

Man bör förebygga att hemligheter läcks genom att separera hemligt innehåll i en appsettings.json till en .Development.json som läggs till i .gitignore, sedan byter man lösenordet i den nya filen då det gamla lösenordet fortfarande återfinns i gamla commits. Man kan även använda GitHubs Secret Scanning-function + pre-commit hooks vilket automatiskt scannar efter hemligheter och varnar innan de hinner committas.


2. BROKEN ACCESS CONTROL

Här var problemet att en inloggad användare kunde uppdatera och radera andra användares notes. Enligt .md-filen skulle det handla om en GET som var sårbar, men så var alltså icke fallet då den sårbarheten hade fixats i en tidigare commit, men PATCH och DELETE hade lämnats öppna.

En sårbarhet som denna kan ju leda till totalt kaos i t.ex. en app för social media, speciellt om PATCH missbrukas och andra användares poster redigeras till något helt annat än vad de själva hade skrivit. Och extra speciellt om den nya posten innehåller en skadlig kodinjektion av något slag.

Fixen är att lägga till samma validering på alla api-anrop, att man alltså kollar att den inloggade användaren är samma som skrivit posten. Men om man gör det manuellt kan det lätt glömmas, vilket hände i det här fallet. Istället kan man använda auktoriserings-policies som definierar vem som äger vad på ett enda ställe, och sen återanvänds av alla endpoints. Det ska även fångas upp av tester.