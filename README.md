# .github

Organisationsgemensamma filer för `smalandsstenars-mekaniska-verkstad`. Repot är
**publikt** — allt som läggs här, inklusive körningsloggarna från Actions, är
läsbart för vem som helst.

## Heartbeat

`.github/workflows/heartbeat.yml` är observatören utanför huset.

Larmen i OpenObserve utvärderas av samma process som skulle rapportera att den
inte kör, och hypervisorns VM-kontroll har samma problem ett steg ned: den ser
webbmaskinen utifrån men skickar sin iakttagelse till OpenObserve *inuti* den
maskinen. Det behövs alltså något utanför nätet, och GitHub Actions är en tjänst
som redan används. Ett schemalagt jobb var tionde minut, två steg med tre försök
vardera:

1. **Den publika sajten måste svara 200.** Det bevisar hela kedjan från
   internet: maskinen är uppe, Docker Swarm är uppe, `proxy-public` avslutar TLS
   och en webbcontainer renderar. OpenObserve kör på samma maskin, så faller det
   här steget kör larmen därinne ändå inte. Att kontrollera sajten och inte
   OpenObserve är en medveten approximation; det som inte fångas är att just
   OpenObserve-containern lagt sig medan resten lever.
2. **Övervakningens vhost måste svara något.** Vilken HTTP-status som helst
   godkänns, inklusive den 403 som allow-listan i kanten ger en runner på
   internet — beviset är litet men äkta: `proxy-public` lever och serverar
   fortfarande vhosten. Bara en anslutning som aldrig kommer upp failar.

GitHub mejlar den som äger workflowet när en schemalagd körning misslyckas
(Settings → Notifications → Actions, *Send notifications for failed workflows
only*).

### Adresserna ligger utanför filen

Övervakningens vhost har med flit ingen publik DNS-post. Att skriva namnet i en
publik fil vore att betala den obskyriteten för ingenting.

| Namn | Typ | Betydelse |
|------|-----|-----------|
| `HEARTBEAT_SITE_URL` | variable | Publika sajten som hämtas, t.ex. `https://www.example.com/` |
| `HEARTBEAT_EDGE_HOST` | **secret** | Övervakningens vhost på samma kant |

Vhosten är en secret och inte en variable, och det är inte kosmetika: runnern
skriver ut varje stegs `env`-block *innan* steget kör, så ett `::add-mask::` i
första steget kommer för sent för sitt eget block och värdet står i klartext i
den publika loggen. Secrets registreras som mask redan vid jobbstart och blir
`***` överallt. Den publika sajten har inget att dölja och får förbli en
variable, där värdet går att läsa tillbaka i UI:t.

Saknas någon av dem failar jobbet direkt med vilken det gäller, i stället för att
`curl` mot en tom sträng.

### Varför `--resolve` i steg 2

Vhosten går inte att slå upp från internet, så en runner kommer aldrig ens fram
till allow-listan — `curl` faller på DNS och steget skulle bli rött varje
körning. I stället slås den publika sajtens adress upp och vhosten tvingas till
samma IP med `curl --resolve`. SNI och `Host` bär namnet, wildcard-certifikatet
täcker det, och nginx svarar.

Undantas `GET /healthz` från allow-listan i `valentis-proxy` — den returnerar
`{"status":"ok"}` och ingen data — blir steget en riktig kontroll av OpenObserve
självt, och då täcks även fallet som steg 1 medger att det missar.

### Två saker att veta om GitHubs schemaläggning

Körningar är *best effort* och skjuts upp under last, och **ett schema stängs av
automatiskt i ett publikt arkiv utan aktivitet på 60 dagar**. Det här är alltså
ett dödmansgrepp för avbrott som varar tiotals minuter, inte en latensmätare.

Regeln är dokumenterad för publika arkiv — vilket det här är — och bara nya
commits nollställer räknaren; en manuell körning eller en release-tagg räcker
inte. I ett arkiv som det här, som inte är tänkt att ha någon aktivitet framåt,
är det själva felläget: GitHub mejlar en varning några dagar i förväg, och
därefter måste schemat slås på för hand igen. En heartbeat som tystnar av sig
själv efter två månader är värre än ingen alls, så antingen hålls repot vid liv
med en commit, eller så ligger workflowet i ett privat arkiv där regeln inte
gäller.
