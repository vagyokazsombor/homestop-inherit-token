# HomeStop – Postman session teszt

A collection automatikusan belép, és a session tokent minden kéréshez hozzáadja. Kézzel nem kell tokent másolni.

## Beállítás

### Variables (collection → Variables fül)
| Változó | Érték | Megjegyzés |
|---|---|---|
| `baseUrl` | `https://homestop.dev` | minden kérés URL-je erre épül |
| `email` | saját email | kézzel töltöm ki |
| `password` | saját jelszó | kézzel töltöm ki |
| `token` | üres | a pre-request script tölti ki |

Az emailt és a jelszót csak a *Current value*-ba írom. Az *Initial value* bekerül az exportba, a Current value csak a gépemen marad.

### Authorization (collection → Authorization fül)
Típus: **API Key**. Ezzel a collection minden kéréshez automatikusan hozzáad egy fejlécet:
- Key: `Cookie`
- Value: `better-auth.session_token={{token}}`
- Add to: Header

A `{{token}}` helyére küldéskor a session token kerül. A kérések Authorization fülén *Inherit auth from parent* van kiválasztva, így mind ezt használja, kérésenként nem kell semmit beállítani.

### Pre-request script (collection → Scripts → Pre-request)
Minden kérés előtt lefut:
```javascript
if (pm.collectionVariables.get('token')) return;
const url = pm.collectionVariables.get('baseUrl');
pm.sendRequest({
  url: url + '/api/auth/sign-in/email', method: 'POST',
  header: { 'Content-Type': 'application/json', 'Origin': url },
  body: { mode: 'raw', raw: JSON.stringify({ email: pm.collectionVariables.get('email'), password: pm.collectionVariables.get('password') }) }
}, (err, res) => pm.collectionVariables.set('token', res.cookies.get('better-auth.session_token')));
```
1. Ha a `token` már ki van töltve, kilép, és nem lép be újra.
2. Ha üres, elküld egy belépési kérést: `POST /api/auth/sign-in/email`, a body-ban az email és a jelszó.
3. Az `Origin` fejléc kötelező, mert a Better Auth enélkül elutasítja a kérést.
4. A szerver a válasz `Set-Cookie` fejlécében küldi a `better-auth.session_token` sütit. A script ezt elmenti a `token` változóba.

## Tesztek

### 1. Get session
`GET {{baseUrl}}/api/auth/get-session`
A Better Auth végpontja. A token alapján visszaadja az aktuális sessiont és a usert.
A teszt: státusz 200, és a `user.email` egyezik a megadott emaillel.

### 2. App – user data
`GET {{baseUrl}}/_serverFn/3da60f48f265def6cb6346d8622d1cb8f6ad9e6571883920b0332f0222ebbfce`

A HomeStop szerverfüggvénye, ami a profilbeállításokat hívja le: locale, user, profile badge, nyelv.

A teszt: státusz 200, és a válaszban szerepel a saját emailem. Ez bizonyítja, hogy nem csak az auth, hanem az alkalmazás is elfogadja a tokent.
A válasz szerializált formátumú, nem sima JSON, ezért a teszt `text()`-et használ.

**Kötelező fejléc:** `Origin: {{baseUrl}}`. Enélkül 403 jön, mert a szerverfüggvényeken CSRF-védelem van. (A `Sec-Fetch-Site: same-origin` is működik.)

## Futtatás
1. Import → `HomeStop.postman_collection.json`
2. Variables: `email` és `password` → *Current value* → Ctrl+S
3. Az *App – user data* kérésnél ellenőrzöm az `Origin` fejlécet.
4. *Run collection*, vagy Send kérésenként
5. A *Test Results* fülön mindkét teszt PASS.

## Megjegyzések
- Az első futásnál a belépés aszinkron, így ha elbukik, elég újra futtatni.
- 403 esetén hiányzik az `Origin`, 404 esetén megváltozott a `_serverFn` hash (az újat a DevTools Network fülén lehet megkeresni).
- Új belépéshez kiürítem a `token`-t. Feltöltés előtt a `password`-öt és a `token`-t is.
