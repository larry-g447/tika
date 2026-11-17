# Tâche 3 - IFT3913: Automatisation des tests de sécurité

---

## 1. Modification du workflow Github Actions

Nous avons modifié le fichier YAML du workflow Github Actions [main-jdk17-build](/.github/workflows/main-jdk17-build.yml) puisqu'il contenait toutes les étapes nécéssaires à la réalistation de ce travail (build maven), mais également puisqu'il utilise Java 17, qui est largement utilisé.

### Modification #1
```yml
steps:
      - uses: actions/checkout@v6
        with:
          fetch-depth: 0
```
Cette modification permet à Github d'aller chercher non seulement le commit qui vient d'être ajouté, mais toute l'historique des commits nécéssaires pour Semgrep. On veut que Semgrep puisse comparer le commit présent à une ancienne version, le paramètre `--baseline-commit`, pour pouvoir détecter de nouvelles vulnérabilité (en ignorant celles déjà présentes avant l'implémentation) Par défaut, Github utilise `fetch-depth: 1`. Cependant, utiliser cette valeur ne permet pas à Github de récupérér `--baseline-commit`, donc Semgrep est incapable de faire la comparaison correctement. 

### Modification #2
```yml
- name: Set up Python
    uses: actions/setup-python@v6
    with:
        python-version: '3.14'

- name: Install Semgrep
    run: pip install semgrep
```
Ces lignes permettent d'installer python dans l'environnement, qui sera utilisé pour installer Semgrep avec pip.

### Modification #3
```yml
- name: Run Semgrep
    env:
        BASELINE: ${{ github.event_name == 'pull_request' && github.event.pull_request.base.sha || github.event.before }} 
    run: | 
        semgrep scan \
        --config auto \
        --baseline-commit $BASELINE  \
        --error
```
Ces lignes configurent et exécutent l'analyse avec Semgrep. Premièrement, la section `env` définie une variable d'environnement nommée `BASELINE`. La valeur de cette variable va dépendre du type d'évènement qui a déclenché l'exécution dur workflow, soit une pull-request ou un push. La variable prendra la valeur du SHA qu'il faudra utiliser pour identifer le commit de base auquel Semgrep va comparer le nouveau commit. Semgrep sera ensuite exécuté avec la commande `semgrep scan`, avec les paramètres suivant:

- `--config-auto`: Permet à Semgrep de choisir automatiquement la configuration selon le projet.
- `--baseline commit $BASELINE`: Compare les résultats de l'analyse actuelle avec l'analyse de base.
- `--error`: Échoue si de nouvelles vulnérabilités sont détectées dans le nouveau commit.

## 2. Introduction de 7 vulnérabilités.

Nous avons créé le fichier [VulnerabilityTests.java] (/tika-core/src/main/java/org/apache/tika/VulnerabilityTests.java) ou nous avons introduit plusieurs vulnérabailités potentielles pour que Semgrep les détecte. ous avons choisi de regrouper toutes les vulnérabilités dans ce fichier pour facilitier l'organisation et la'identification de ces vulnérabilités dans le rapport Semgrep. Certaines n'ont pas été détectées, comme la suivante: 

```java
     // 6. Hardcoded Credentials
    private static final String API_KEY =
            "AKIAIOSFODNN7EXAMPLE_SECRET_KEY";

    public String getApiKey() {
        return API_KEY;
    }
```
Cette vulnérabilité est très connue et pose des risques de sécurités très grand dans des applications à grande échelle. Cependant, Semgrep n'a pas réussi à la détecter. Cela pourrait être causé par le motif de la clé que nous avons choisi, qui ne correspond peut-être pas aux motifs règles utilisés par Semgrep.

Nous sommes finalement arrivés à trouver 7 vulnérabilités détectées par Semgrep, qui sont les suivantes:

## Documentation et justification des vulnérabilités introduites

### 1. Désérialisation non sécurisée (Insecure Deserialization)

**Type de vulnérabilité :** CWE-502 – Deserialization of Untrusted Data.

**Code introduit :** La méthode `deserialize()` utilise la classe `ObjectInputStream` et sa méthode `readObject()` pour reconstruire un objet Java à partir d'un tableau de bytes.

**Pourquoi le code est vulnérable :** La désérialisation d'objets provenant d'une source non fiable peut entraîner des comportements inattendus lors de la reconstruction des objets. Certains objets peuvent déclencher des opérations dangereuses pendant leur désérialisation. Le code ne vérifie ni la provenance ni la sécurité des données avant de les désérialiser. Cette pratique peut donc permettre à des données manipulées de provoquer des conséquences indésirables dans l'application.

**Résultat obtenu avec Semgrep :**
- Règle détectée : `java.lang.security.audit.object-deserialization.object-deserialization`
- Ligne détectée : 23-24.

### 2. Utilisation d'un algorithme cryptographique faible (MD5)

**Type de vulnérabilité :** CWE-328 – Use of Weak Hash.

**Code introduit :** La méthode `calculateHash()` utilise `MessageDigest.getInstance("MD5")` pour calculer le condensat d'une chaîne de caractères.

**Pourquoi le code est vulnérable :** MD5 est un algorithme de hachage obsolète qui présente des faiblesses cryptographiques, notamment en matière de résistance aux collisions. Cela signifie que deux entrées différentes peuvent produire le même hash (collision). 

**Résultat obtenu avec Semgrep :**
- Règle détectée : `java.lang.security.audit.crypto.use-of-md5.use-of-md5`
- Ligne détectée : 32.

Semgrep nous propose également d'utiliser HMAC au lieu de MD5.

### 3. Injection XML externe (XML External Entity Injection – XXE)

**Type de vulnérabilité :** CWE-611 – Improper Restriction of XML External Entity Reference.

**Code introduit :** La méthode `createXmlParser()` crée une instance de `DocumentBuilderFactory` et configure la fonctionnalité `disallow-doctype-decl` à `false`.

**Pourquoi le code est vulnérable :** Cette configuration autorise les déclarations DOCTYPE dans les documents XML. Si le parseur traite un document XML provenant d'une source non fiable, celui-ci peut contenir des références à des entités externes. Le traitement de ces entités peut exposer des ressources locales ou provoquer des requêtes vers des ressources externes. Le parseur n'est donc pas configuré de manière assez sécurisée contre les attaques XXE.

**Résultat obtenu avec Semgrep :**
- Règle détectée : `java.lang.security.audit.xxe.documentbuilderfactory-disallow-doctype-decl-false.documentbuilderfactory-disallow-doctype-decl-false`
- Lignes détectées : 42-44.

### 4. Injection SQL (SQL Injection)

**Type de vulnérabilité :** CWE-89 – Improper Neutralization of Special Elements used in an SQL Command.

**Code introduit :** La méthode `findUser()` construit une requête SQL en concaténant directement la variable `username` à la chaîne contenant la requête. Elle exécute ensuite cette requête à l'aide d'un objet `Statement`.

**Pourquoi le code est vulnérable :** Les données fournies à la méthode sont intégrées directement dans la requête SQL sans être séparées de la commande. Une entrée contenant des caractères ayant une signification particulière pour SQL pourrait ainsi modifier la structure de la requête. Cela pourrait permettre de l'injection SQL (l'exécution d'opérations SQL hors du programme) par un acteur externe.

**Résultat obtenu avec Semgrep :**
Deux règles différentes ont détecté cette vulnérabilité :
- Règle 1 : `java.lang.security.audit.formatted-sql-string.formatted-sql-string`
- Règle 2 : `java.lang.security.audit.sqli.jdbc-sqli.jdbc-sqli`

Semgrep nous propose que l'utilisation d'une requête préparée avec `PreparedStatement` permettrait de séparer les données des instructions SQL.
### 5. Utilisation du mode de chiffrement AES-ECB

**Type de vulnérabilité :** CWE-327 – Use of a Broken or Risky Cryptographic Algorithm.

**Code introduit :** La méthode `encryptWithAES_ECB()` utilise `Cipher.getInstance("AES/ECB/PKCS5Padding")` pour chiffrer des données avec l'algorithme AES en mode ECB.

**Pourquoi le code est vulnérable :** Le mode ECB chiffre chaque bloc de données indépendamment des autres. Par conséquent, deux blocs contenant les mêmes données produisent le même résultat chiffré. Cette propriété peut révéler des motifs présents dans les données originales. 


**Résultat obtenu avec Semgrep :**
Deux règles différentes ont détecté cette vulnérabilité :
- Règle 1 : `java.lang.security.audit.crypto.ecb-cipher.ecb-cipher`
- Règle 2 : `java.lang.security.audit.crypto.use-of-aes-ecb.use-of-aes-ecb`
- Ligne détectée : 67.

Selon Semgrep, ce mode ne fournit pas de mécanisme intégré de vérification de l'intégrité des données. Il est donc déconseillé pour protéger des informations sensibles. Un mode de chiffrement authentifié (comme AES-GCM) serait plus approprié.

### 6. Utilisation de l'algorithme de hachage SHA-1

**Type de vulnérabilité :** CWE-328 – Use of Weak Hash.

**Code introduit :** La méthode `calculateSHA1()` utilise `MessageDigest.getInstance("SHA-1")` pour calculer le condensat d'une chaîne de caractères.

**Pourquoi le code est vulnérable :** SHA-1 est un algorithme de hachage qui présente des faiblesses cryptographiques, comme la faible résistance aux collisions. Il n'est plus considéré comme suffisamment sûr pour les usages nécessitant une fonction de hachage cryptographique fiable. Son utilisation peut compromettre la sécurité des mécanismes qui reposent sur l'unicité des condensats..

**Résultat obtenu avec Semgrep :**
- Règle détectée : `java.lang.security.audit.crypto.use-of-sha1.use-of-sha1`
- Ligne détectée : 79.

Semgrep propose l'utilisation PBKDF2 pour l'encryption des mots de passe, et SHA256 ou SHA512 pour d'autres applications.

### 7. Communication réseau non chiffrée (Unencrypted Network Communication)

**Type de vulnérabilité :** CWE-319 – Cleartext Transmission of Sensitive Information.

**Code introduit :** La méthode `openUnencryptedConnection()` utilise la classe `Socket` de Java pour établir une connexion réseau standard avec un hôte et un port donnés.

**Pourquoi le code est vulnérable :** Un `Socket` standard ne chiffre pas les données échangées. Les communications transitent donc sans protection cryptographique au niveau de la connexion. Si des informations sensibles sont transmises, elles peuvent être exposées à une interception sur le réseau. 

**Résultat obtenu avec Semgrep :**
- Règle détectée : `java.lang.security.audit.crypto.unencrypted-socket.unencrypted-socket`
- Ligne détectée : 89.

Une connexion sécurisée utilisant TLS (avec par exemple `SSLSocket`) permettrait de protéger les communications.