# TP1-SOA-24152

## Étape 1 — Créer et lancer le service

1. `@WebService` expose la classe `EmpruntService` en tant que service web. `@WebFault` associe l'exception `LivreIndisponibleException` au fault déclaré `LivreIndisponible`.
2. JAXB utilise le constructeur sans argument pour instancier les classes de données lors de la désérialisation XML, puis renseigne leurs champs.

## Étape 2 — Lire le WSDL généré

1. Le port `EmpruntPort` du service `EmpruntService` est publié à l'adresse `http://localhost:8080/bib/EmpruntService`. Le `portType` définit les opérations suivantes.

   | Opération | Message d'entrée | Message de sortie |
   | --- | --- | --- |
   | `emprunterLivre` | `emprunterLivre` | `emprunterLivreResponse` |
   | `rendreLivre` | `rendreLivre` | `rendreLivreResponse` |

2. Le binding utilise SOAP 1.1 avec le style `document` et le mode `literal`.
3. Le XSD importé par le WSDL définit `isbn` et `dateRetour` en `xs:string`. L'expression régulière de l'ISBN est vérifiée dans le code Java et n'apparaît pas dans le XSD.
4. Dans la requête générée, les paramètres locaux `idAdherent` et `isbn` sont non qualifiés. L'élément d'opération `emprunterLivre` appartient à l'espace de noms `http://supnum.mr/bib`.
5. Dans la vue Raw, la ligne `POST /bib/EmpruntService HTTP/1.1` et les en-têtes relèvent de HTTP, qui transporte la requête et la réponse. Après les en-têtes, XML fournit la syntaxe du contenu. SOAP définit la structure de ce contenu avec `Envelope`, `Header` et `Body`, ainsi que le format des Faults. Le WSDL décrit l'adresse, les opérations, les messages, les types et le binding du service.

## Étape 3 — Envoyer des messages SOAP

1. Le binding SOAP sur HTTP utilise `POST` pour envoyer une enveloppe dans le corps de la requête. L'opération est indiquée par le message SOAP, même si elle ne fait que lire des données.
2. Le jeton n'empêche pas le traitement. Le bloc est facultatif en l'absence de `soap:mustUnderstand` et le service peut l'ignorer. Sur un serveur fraîchement démarré, le message valide renvoie HTTP 200 et le numéro `E-2026-0812`.
3. Avec le même `Content-Type: text/xml`, l'enveloppe SOAP 1.2 est rejetée avec une Fault `VersionMismatch` (HTTP 500). Le binding attend l'espace de noms d'enveloppe de SOAP 1.1.
4. Les cinq corrections sont les suivantes.

   | Erreur | Correction |
   | --- | --- |
   | La balise d'ouverture écrit `soap:envelope` en minuscules | Écrire `soap:Envelope` |
   | Le préfixe `bib` est associé à `http://supnum.mr/bibliotheque` | Utiliser `http://supnum.mr/bib` |
   | La balise fermante de l'opération omet le préfixe | Écrire `</bib:rendreLivre>` |
   | Le paramètre est écrit `bib:numero` | Écrire `<numeroEmprunt>E-2026-0812</numeroEmprunt>` |
   | Le `Header` suit le `Body` | Placer le `Header` avant le `Body` |

   Les balises mal appariées rendent le XML mal formé. Avec un XML bien formé, un espace de noms d'opération incorrect empêche le routage. Avec le `Header` seul mal placé, Metro traite le `Body` et renvoie HTTP 200 avec une pénalité de 0, car il ne contrôle pas la structure complète de l'enveloppe à ce stade. Sans validation du schéma, Metro peut ignorer le paramètre inconnu et transmettre `null` à `rendreLivre`. À l'étape 3, la méthode renvoie alors zéro. Après l'étape 4, elle déclenche une Fault `Client`.

## Étape 4 — Provoquer et construire des Faults

1. L'ISBN `978-2-10` renvoie `Client` et `ISBN mal formé : 978-2-10` avec HTTP 500. L'ISBN `978-2-10-074444-4` renvoie `Server` et `Livre indisponible` avec HTTP 500.
2. Le `detail` contient l'élément `LivreIndisponible` avec `isbn` égal à `978-2-10-074444-4` et `retourPrevu` égal à `2026-12-02`. Le `wsdl:fault` de l'opération référence le message `LivreIndisponibleException`, dont la partie pointe vers cet élément. Le XSD importé dans `wsdl:types` définit la structure de l'élément.
3. JAX-WS choisit `Server` pour l'exception métier déclarée. La requête est valide et l'état du stock empêche de la satisfaire. Je garderais `Server` avec un détail métier qui indique quand le livre sera disponible.
4. Avec `soap:mustUnderstand="1"`, le serveur renvoie une Fault `MustUnderstand` avant d'appeler l'opération. Avec `"0"`, il ignore le jeton et traite la requête.
5. En SOAP 1.2, `soap:Code/soap:Value` contient `soap:Sender` à la place de `faultcode` avec `Client`. `soap:Reason/soap:Text` porte le message et sa langue avec `xml:lang="fr"`. `soap:Detail` remplace `detail`.

## Étape 5 — Générer le code depuis le WSDL

1. L'opération est `emprunterLivre`, son message de fault est `bib:LivreIndisponibleFault` et la valeur de `SOAPAction` est `"http://supnum.mr/bib/emprunterLivre"`.
2. La signature générée est `EmpruntType emprunterLivre(String idAdherent, String isbn) throws LivreIndisponibleFault`.
3. La date de retour est générée en `javax.xml.datatype.XMLGregorianCalendar` à partir de `xs:date`. Dans la partie A, elle est déclarée en Java comme `String`.
4. La classe générée est `LivreIndisponibleFault`, d'après le nom du `wsdl:message` référencé par le `wsdl:fault`.

## Étape 6 — Implémenter et comparer

1. Metro publie le contrat fourni dans `wsdl/EmpruntService.wsdl`. Il conserve la restriction `xs:pattern` d'`IsbnType` et les dates de type `xs:date`.
2. Pour la partie B, il faut qualifier `idAdherent` et `isbn` en écrivant `bib:idAdherent` et `bib:isbn` dans la requête. Le schéma de la partie B déclare `elementFormDefault="qualified"`. Il faut aussi envoyer l'action SOAP du contrat. Le fichier `01-emprunter.xml` du rendu garde la forme demandée à l'étape 3 pour la partie A.
3. Sans `@SchemaValidation`, l'ISBN `978-2-10` arrive à l'implémentation comme une chaîne et l'emprunt est accepté (HTTP 200). Avec l'annotation, Metro rejette le message avant l'appel Java et renvoie une Fault `Client` (HTTP 500). Le message `cvc-pattern-valid` signale la violation du motif d'`IsbnType`.
4. Sans `wsdlLocation`, Metro génère le WSDL à partir de l'interface et des classes issues de `wsimport`. Le WSDL généré importe un XSD séparé. Ce schéma déclare l'ISBN en `xs:string` et perd la restriction `IsbnType`. Le type `xs:date` est conservé grâce au mapping JAXB de la date.

## Conclusion

1. Le WSDL de la partie A expose en plus l'opération `rendreLivre`. Son XSD utilise `xs:string` pour la date de retour, tandis que le WSDL de la partie B utilise `xs:date`.
2. En code-first, renommer le champ public `dateRetour` modifie le schéma généré. Les clients qui attendent l'ancien nom risquent d'échouer à la validation ou de ne plus lire la date. En contract-first, le WSDL fourni reste la source du contrat et l'implémentation doit continuer à respecter les types générés.
3. Pour Paiement, je choisirais le contract-first afin que Scolarité (.NET), Bibliothèque (PHP) et le portail (Node.js) aient un contrat XML commun avant de développer leurs clients. Pour un prototype interne de deux jours, je choisirais le code-first pour obtenir rapidement un service testable.
