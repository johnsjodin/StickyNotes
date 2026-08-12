1. SENSITIVE DATA EXPOSURE

Problemet var att lösenordet till en test-användare låg i klartext i appsettings.json, vilket följde med in i git-repot och var öppet och tillgängligt.

Ofta när man använder en testanvändare under utveckligen så har den användaren förhöjda rättigheter, vilket gör att ett läckt lösenord till en sån användare kan ställa till mycket oreda i händerna på fel person. Lösenord brukar även återanvändas, vilket gör att läckta lösenord även jämförs mot andra servrar och tjänster vilket riskerar en ännu större säkerhetsläcka och möjlighet till sabotage.

Man bör förebygga att hemligheter läcks genom att separera hemligt innehåll i en appsettings.json till en .Development.json som läggs till i .gitignore, sedan byter man lösenordet i den nya filen då det gamla lösenordet fortfarande återfinns i gamla commits. Man kan även använda GitHubs Secret Scanning-function + pre-commit hooks vilket automatiskt scannar efter hemligheter och varnar innan de hinner committas.


2. 