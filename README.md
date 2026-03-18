# LAB5-Reverse-Engineering-de-UnCrackable-Level-2

Pour ce lab, l'analyse a été réalisée entièrement en analyse statique, sans émulateur.L'APK ne se lance pas sur l'émulateur.

# Partie 2 — Trouver où commence la vérification

Étape 3 — Décompiler l’APK avec JADX

1. Le lancement de l'APK UnCrackable-Level2.apk dans JADX:

<img width="963" height="56" alt="image" src="https://github.com/user-attachments/assets/f9ff92f3-ce61-4126-841c-0f55b419571a" />

2. L'observation de la classe MainActivity:

L'arborescence des classes s'affiche dans le panneau gauche. J'ai navigué vers le package sg.vantagepoint.uncrackable2 et ouvert la classe MainActivity.

<img width="1920" height="1030" alt="image" src="https://github.com/user-attachments/assets/60b13e1d-e8d7-4b4b-a180-bb63649deef2" />

Étape 4 — Repérer l'appel de validation dans MainActivity

Dans la méthode verify(), j'observe que la chaîne saisie par l'utilisateur est récupérée puis transmise à la méthode a() d'un objet m de type CodeCheck :

<img width="1467" height="465" alt="image" src="https://github.com/user-attachments/assets/5a473a0f-ec59-45b0-877c-84dcdd5a6257" />

La vérification n'est donc pas effectuée dans MainActivity elle-même, mais déléguée à la classe CodeCheck.


# Partie 3 — Comprendre le rôle de CodeCheck


Étape 5 — Identifier la classe qui effectue la vérification

 Dans JADX, j'ai ouvert la classe CodeCheck dans le package sg.vantagepoint.uncrackable2. Elle contient deux éléments essentiels :
Une méthode déclarée native :

> javaprivate native boolean bar(byte[] bArr);

Une méthode a() qui l'appelle :

>javapublic boolean a(String str) {
>    return bar(str.getBytes());
>}

La chaîne saisie par l'utilisateur est convertie en tableau d'octets puis passée à bar(). Le mot-clé native indique que cette méthode est implémentée non pas en Java, mais dans une bibliothèque native (C/C++).

Note : Dans ce cas, System.loadLibrary("foo") est absent de CodeCheck — il se trouve dans MainActivity. Le résultat est le même : c'est la bibliothèque libfoo.so qui contient la vraie logique de vérification.

<img width="866" height="332" alt="image" src="https://github.com/user-attachments/assets/810bd3a0-6cfc-4f8f-83a3-361358317155" />

Checkpoint ✅ : La vérification se fait dans une bibliothèque native. Il faut maintenant analyser libfoo.so.

# Partie 4 — Retrouver la bibliothèque native

Étape 6 — Extraire le contenu de l’APK

unzip est une commande Linux. Sur Windows on utilise Expand-Archive à la place — c'est l'équivalent.

Un APK étant un fichier ZIP renommé, je l'ai copié avec l'extension .zip puis extrait avec Expand-Archive. Le dossier lib contient 4 variantes de libfoo.so selon l'architecture :


<img width="962" height="938" alt="image" src="https://github.com/user-attachments/assets/2d73c4b7-59d3-4160-af5f-0a7790676912" />

<img width="836" height="296" alt="image" src="https://github.com/user-attachments/assets/619f3239-dab5-40dc-9a5f-77b9ea3ef203" />

Dossier lib: 

<img width="768" height="195" alt="image" src="https://github.com/user-attachments/assets/e2eaa71c-1240-4a77-9610-a73de44b2d2b" />

Pour l'analyse statique, j'utiliserai la version x86 :

la bibliothèque libfoo.so

<img width="797" height="98" alt="image" src="https://github.com/user-attachments/assets/eb915ce2-82c3-4775-a161-f3bee1f5adb0" />

# Partie 5 — Analyser le code natif avec Ghidra

Étape 7 — Importer libfoo.so dans Ghidra

Lancement de Ghidra: 

<img width="963" height="38" alt="image" src="https://github.com/user-attachments/assets/2633ecfe-95cc-44c3-bb10-6440c36beb28" />

<img width="633" height="977" alt="image" src="https://github.com/user-attachments/assets/c79c09b8-a7b7-49e3-850a-74496b8e087d" />


J'ai créé un nouveau projet Ghidra et importé le fichier lib/x86/libfoo.so. Ghidra a automatiquement détecté :

Format : ELF (Executable and Linking Format)
Architecture : x86 32-bit Little Endian
Compilateur : gcc

<img width="981" height="742" alt="image" src="https://github.com/user-attachments/assets/9153c288-1079-476b-a5e1-d601d7815d41" />

<img width="632" height="477" alt="image" src="https://github.com/user-attachments/assets/c0968f2b-9214-4baa-af0f-7811da7961db" />

L'inportation de la bibliothèque libfoo.so:

<img width="973" height="736" alt="image" src="https://github.com/user-attachments/assets/0278b089-3edb-4ebb-88af-97312b6bc0a3" />

<img width="612" height="312" alt="image" src="https://github.com/user-attachments/assets/43feca08-d3db-4e85-b3a0-d31007394c05" />

 Un double-clique sur libfoo.so

<img width="1006" height="1026" alt="image" src="https://github.com/user-attachments/assets/53e71a6f-b71f-4cb3-bc79-1c800479032d" />

Lancement de l’analyse automatique

<img width="497" height="161" alt="image" src="https://github.com/user-attachments/assets/9d0fc750-3fa4-4f30-ac9a-448f2fa99057" />

<img width="1232" height="741" alt="image" src="https://github.com/user-attachments/assets/3d9b2711-4178-4b5d-9e27-e317ca57bab1" />

<img width="1915" height="988" alt="image" src="https://github.com/user-attachments/assets/51153d3c-f61b-4585-aead-7e488257298c" />

 la bibliothèque est chargée dans Ghidra.

 Étape 8 — Chercher la fonction JNI liée à bar

Dans le panneau Symbol Tree → Exports, j'ai recherché un symbole contenant CodeCheck_bar et j'ai trouvé :

Java_sg_vantagepoint_uncrackable2_CodeCheck_bar

Ce nom suit la convention JNI standard : Java_ + package + classe + méthode. Il s'agit de l'implémentation native de la méthode bar() déclarée dans CodeCheck.java.

En double-cliquant dessus, le décompilateur affiche le code suivant :

<img width="1407" height="975" alt="image" src="https://github.com/user-attachments/assets/d3753161-db40-4cab-9596-24cb1e5c5c2c" />

Checkpoint ✅ : La fonction native de vérification est identifiée dans libfoo.so.


# Partie 6 — Comprendre la comparaison avec strncmp

Étape 9 — Lire le pseudo-code de la fonction native

Dans le décompilateur Ghidra, j'observe que la fonction copie une chaîne dans un buffer local puis la compare à l'entrée utilisateur via strncmp :

<img width="695" height="278" alt="image" src="https://github.com/user-attachments/assets/79d5438f-d30c-4b4b-9a28-1ea4e8f3eff0" />

La chaîne secrète est visible en clair directement dans le pseudo-code : **`Thanks for all the fish`**

Checkpoint ✅ : La chaîne cachée utilisée comme secret est identifiée directement.

Étape 10 — Retrouver la valeur comparée

Dans ce cas, la chaîne est visible en clair dans le décompilateur Ghidra, ce qui rend l'étape 10 directement résolue.

La valeur hexadécimale correspondante est :

6873696620656874206c6c6120726f6620736b6e616854

Ce qui correspond octet par octet à la chaîne ASCII "Thanks for all the fish"

# Partie 7 — Décoder le secret

Étape 11 — Convertir l’hexadécimal en ASCII

1. Décodage hexadécimal de la chaine dans Python:

<img width="917" height="182" alt="image" src="https://github.com/user-attachments/assets/5b0d61c5-2bfc-495f-b540-e212843ca830" />

2. Résultat ASCII obtenu:

<img width="513" height="52" alt="image" src="https://github.com/user-attachments/assets/009e9959-52a8-4145-96e4-f187f0a7f4ec" />

Étape 12 — Inverser la chaîne pour obtenir la phrase finale

1. Inversion de la chaîne dans Python:

<img width="628" height="42" alt="image" src="https://github.com/user-attachments/assets/54fa6298-7743-40c7-b5cf-b942b00a4011" />

2. Secret final obtenu:

<img width="507" height="58" alt="image" src="https://github.com/user-attachments/assets/f497ba0f-6c01-4de0-ba86-04ce4c024e6b" />

# Partie 8 — Valider la solution

Étape 13 — Tester le secret dans l’application

L'analyse statique complète a permis de retrouver le secret sans exécuter l'application :

MainActivity transmet la saisie à CodeCheck.a()

CodeCheck.a() appelle la méthode native bar() via JNI

libfoo.so compare la saisie à la chaîne "Thanks for all the fish" via strncmp

La chaîne était stockée en hexadécimal inversé dans la bibliothèque native

Le secret final est : Thanks for all the fish

La validation dans l'application n'a pas été effectuée (pas d'émulateur), mais l'analyse statique confirme sans ambiguïté que cette chaîne est la bonne réponse.

Checkpoint final ✅ : Le challenge UnCrackable Level 2 est résolu par analyse statique.






















