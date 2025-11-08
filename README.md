# Laboratorul 04: Configurarea unui pipeline CI/CD cu Jenkins

## Descrierea Proiectului

Proiectul utilizat pentru acest pipeline este o aplicație PHP minimală creată special pentru acest laborator. Este găzduită pe GitHub la [https://github.com/CDenis7/my-php-project](https://github.com/CDenis7/my-php-project).

<img width="974" height="459" alt="image" src="https://github.com/user-attachments/assets/1ec17c5f-c539-4620-bbd6-11421aeebce3" />

Aplicația constă într-o clasă simplă `src/Calculator.php` cu o metodă `add`. Un fișier corespunzător `tests/CalculatorTest.php` folosește `PHPUnit` pentru a verifica dacă metoda `add` funcționează corect.

Repository-ul conține, de asemenea:
* `composer.json` pentru a defini dependențele (în mod specific `phpunit/phpunit`).
* `phpunit.xml.dist` pentru a configura rularea testelor.
* `Jenkinsfile` pentru a defini pașii pipeline-ului CI/CD pentru Jenkins.

Scopul acestei configurări este ca Jenkins să preia automat codul, să instaleze dependențele și să ruleze testele unitare de fiecare dată când este declanșat pipeline-ul.

## Pași pentru Configurarea Jenkins Controller

1.  **Crearea `docker-compose.yml`:** Un fișier `docker-compose.yml` a fost creat pentru a defini mediul Jenkins.

<img width="720" height="927" alt="image" src="https://github.com/user-attachments/assets/0c39cc85-b53d-4a74-8a44-3764b5a9fe31" />

2.  **Definirea Serviciului Controller:** Serviciul `jenkins-controller` a fost definit folosind imaginea `jenkins/jenkins:lts`.
3.  **Configurarea Porturilor:** Porturile `8080:8080` (pentru interfața web) și `50000:50000` (pentru comunicarea cu agenții) au fost expuse.
4.  **Adăugarea Volumului:** Un volum numit `jenkins_home` a fost montat pe `/var/jenkins_home` pentru a asigura persistența datelor (plugin-uri, job-uri, configurare) la repornirea containerului.
5.  **Adăugarea Rețelei:** O rețea de tip "bridge" numită `jenkins-network` a fost creată pentru a permite controller-ului și agentului să comunice.
6.  **Pornirea Containerului:** Controller-ul a fost lansat folosind `docker-compose up -d`.

<img width="974" height="378" alt="image" src="https://github.com/user-attachments/assets/006c2568-d122-49d6-8098-08a12f5f5f3a" />

7.  **Configurarea Inițială:**
    * Parola inițială de administrator a fost preluată din log-urile containerului folosind `docker logs jenkins-controller`.

<img width="974" height="449" alt="image" src="https://github.com/user-attachments/assets/54d475c1-8e37-47ff-b74d-f8c25d0f8563" />

  * Expertul de configurare (setup wizard) a fost finalizat instalând setul "Install suggested plugins", care a inclus plugin-urile necesare `git` și `ssh-agent`.
  * A fost creat un nou utilizator administrator.

<img width="974" height="443" alt="image" src="https://github.com/user-attachments/assets/83bcb85c-37b3-4e45-bccf-ebfe64c45f56" />

## Pași pentru Configurarea Agentului SSH

1.  **Generarea Cheilor SSH:** A fost creat un director `secrets` și a fost generată o pereche de chei SSH (`jenkins_agent_ssh_key` și `jenkins_agent_ssh_key.pub`) folosind `ssh-keygen`.

<img width="974" height="382" alt="image" src="https://github.com/user-attachments/assets/19907823-4156-4945-b9de-7816c3fb9409" />

2.  **Crearea `Dockerfile`-ului:** Un `Dockerfile` a fost creat având la bază imaginea `jenkins/ssh-agent`. Acest fișier a fost actualizat de mai multe ori pentru a instala toate dependențele necesare proiectului PHP. `Dockerfile`-ul final instalează `php-cli`, `wget`, `php-curl`, `php-xml`, `php-mbstring`, `unzip`, `php-zip` și `composer`.

<img width="1262" height="298" alt="image" src="https://github.com/user-attachments/assets/742a0bb6-1808-4959-b610-5b1daf3dd6a7" />

3.  **Actualizarea `docker-compose.yml`:** Serviciul `ssh-agent` a fost adăugat în fișierul `docker-compose.yml`.
    * A fost configurat să se construiască (`build`) folosind `Dockerfile`-ul local.
    * Variabila de mediu `JENKINS_AGENT_SSH_PUBKEY` a fost transmisă dintr-un fișier `.env`.
    * A fost adăugat la `jenkins-network` pentru a comunica cu controller-ul.
    * A fost adăugată o proprietate `depends_on: [jenkins-controller]`.
4.  **Crearea Fișierului `.env`:** Conținutul cheii publice (`jenkins_agent_ssh_key.pub`) a fost copiat într-un fișier `.env`.

<img width="1328" height="116" alt="image" src="https://github.com/user-attachments/assets/4b2074f0-abf2-43f8-b99b-5dce63336ff7" />

5.  **Construire și Pornire:** Întregul stack a fost lansat cu `docker-compose up -d --build`, comandă care a construit imaginea personalizată a agentului și a pornit ambele containere.

## Pași pentru Crearea și Configurarea Pipeline-ului Jenkins

1.  **Adăugarea Credențialelor Jenkins:**
    * În interfața Jenkins, am navigat la **Manage Jenkins > Manage Credentials**.
    * A fost adăugată o nouă credențială de tip "SSH Username with private key".
    * **Username** a fost setat la `jenkins`.
    * **Private Key** a fost furnizată prin "Enter directly", lipind conținutul fișierului `secrets/jenkins_agent_ssh_key` (cheia privată).

<img width="974" height="445" alt="image" src="https://github.com/user-attachments/assets/e565deb4-4706-4a3e-9277-ba23a63cc4dd" />

2.  **Adăugarea Nodului (Agent) Jenkins:**
    * Am navigat la **Manage Jenkins > Manage Nodes and Clouds**.
    * A fost creat un **New Node** cu numele `ssh-agent1` și tipul **Permanent Agent**.

<img width="974" height="450" alt="image" src="https://github.com/user-attachments/assets/72317d0f-7232-4ba3-b994-dc8f1b352c61" />
   
  * **Label (Etichetă):** `php-agent` (pentru a fi folosit în `Jenkinsfile`).
    * **Remote root directory:** `/home/jenkins/agent`
    * **Launch method (Metodă de lansare):** "Launch agents via SSH".
    * **Host:** `ssh-agent` (numele serviciului din `docker-compose.yml`).
    * **Credentials:** Am selectat credențialele `jenkins` create în pasul anterior.
    * **Host Key Verification Strategy:** Setat la **"Non-verifying Verification Strategy"** pentru a preveni problemele la reconstruirea containerului agent.
    * Agentul a fost apoi lansat dând clic pe "Launch agent" de pe pagina nodului.

2.  **Crearea `Jenkinsfile`-ului:** Un `Jenkinsfile` a fost adăugat la rădăcina repository-ului `my-php-project`. Acest fișier definește pipeline-ul cu un `agent { label 'php-agent' }` și trei etape (stages):
    * **Checkout Code:** (Folosește pasul `checkout scm`, care este și rulat implicit).
    * **Install Dependencies:** Rulează `sh 'composer install'`.
    * **Test:** Rulează `sh 'vendor/bin/phpunit'`.

<img width="669" height="851" alt="image" src="https://github.com/user-attachments/assets/5858f3a3-9152-45ef-ab85-c9857666f21a" />

<img width="974" height="425" alt="image" src="https://github.com/user-attachments/assets/ef1193ff-db90-41d6-8147-924f3f4f9f36" />

3.  **Crearea Job-ului Jenkins:**
    * Un nou item de tip **"Pipeline"** a fost creat în Jenkins.

   <img width="974" height="447" alt="image" src="https://github.com/user-attachments/assets/6b848f16-e330-4e98-abed-fa4ed8fc147b" />

  * Sub secțiunea **Pipeline**, **Definition** a fost setată la **"Pipeline script from SCM"**.
    * **SCM** a fost setat la **Git**.
    * A fost adăugată **URL-ul Repository-ului** `my-php-project`.
    * **Branch Specifier** a fost schimbat din `*/master` în `*/main`.
    * Pipeline-ul a fost rulat dând clic pe **"Build Now"**, ceea ce a rezultat într-o construcție (build) de succes.
    * 
<img width="974" height="445" alt="image" src="https://github.com/user-attachments/assets/cae826af-328f-46eb-8ef8-a2eda4147002" />

<img width="974" height="444" alt="image" src="https://github.com/user-attachments/assets/73d3e9c7-dcc9-446d-b096-888aea2f493e" />

## Întrebări de Discuție

### 1. Care sunt avantajele utilizării Jenkins pentru automatizarea sarcinilor DevOps?

Jenkins este un server de automatizare open-source puternic, fiind o piatră de temelie a DevOps. Principalele sale avantaje sunt:

* **Automatizare:** Automatizează întregul pipeline "build, test, deploy", eliminând sarcinile manuale, predispuse la erori, mărind viteza și permițând integrarea continuă (CI) și livrarea continuă (CD).
* **Ecosistem Vast de Plugin-uri:** Cu mii de plugin-uri, Jenkins se poate integra cu aproape orice instrument sau tehnologie (de ex., Git, Docker, Kubernetes, AWS, Azure, Jira). Acest lucru îl face incredibil de flexibil și adaptabil.
* **Pipeline as Code (Pipeline-ul ca și cod):** Folosind un `Jenkinsfile`, întregul pipeline CI/CD poate fi definit ca și cod. Acest cod este versionat, reutilizabil și poate fi revizuit, la fel ca și codul aplicației.
* **Build-uri Distribuite:** Jenkins poate distribui sarcinile de build și testare pe mai multe mașini "agent". Acest lucru permite execuția paralelă și scalarea pentru a gestiona proiecte de orice dimensiune, accelerând procesul de build.
* **Open Source și Comunitate:** Fiind gratuit și open-source, are o comunitate masivă. Asta înseamnă că este în permanență îmbunătățit, suportul este larg disponibil și nu este blocat de un anumit furnizor.

### 2. Ce alte tipuri de agenți Jenkins există?

Pe lângă **Agentul SSH** pe care l-am folosit (care este un model "push" în care controller-ul inițiază conexiunea), alte tipuri comune includ:

* **Agenți JNLP (Java Network Launch Protocol):** Acesta este un model "pull". Agentul (care rulează adesea ca un container sau pe un VM separat) se conectează *la* controller-ul Jenkins. Aceasta este o metodă foarte comună pentru agenții care rulează în rețele diferite sau în medii cloud.
* **Agenți Docker (Dinamici):** Jenkins poate fi configurat să pornească dinamic un container Docker nou și curat pentru fiecare build. Aceasta este o metodă foarte populară, deoarece oferă un mediu de build perfect, izolat și reproductibil pentru fiecare job, apoi dispare, economisind resurse.
* **Agenți Kubernetes (Dinamici):** Similar cu Docker, Jenkins poate lansa dinamic "pod-uri" agent în cadrul unui cluster Kubernetes. Acest lucru este extrem de puternic pentru medii de build elastice, la scară largă.
* **Nodul Încorporat (Built-in Node):** Controller-ul Jenkins însuși poate rula build-uri. Acest lucru este în regulă pentru proiecte mici sau pentru testare, dar **nu** este recomandat pentru producție, deoarece build-urile pot consuma toate resursele controller-ului și pot face interfața instabilă.

### 3. Ce probleme ați întâmpinat la configurarea Jenkins și cum le-ați rezolvat?

Acest laborator a implicat o cantitate semnificativă de depanare iterativă. Principalele probleme au fost:

1.  **Eroare la ramura `master` vs. `main`:** Primul build a eșuat cu o eroare Git: `couldn't find remote ref refs/heads/master`.
    * **Soluție:** Acest lucru s-a întâmplat deoarece job-ul Jenkins era configurat implicit să caute o ramură `master`, dar ramura implicită a noului repository GitHub era `main`. Problema a fost rezolvată prin schimbarea **"Branch Specifier"** în configurația job-ului Jenkins la `*/main`.

2.  **`'ssh-agent1' este offline`:** După reconstruirea containerului `ssh-agent` (de exemplu, pentru a adăuga software nou), interfața Jenkins arăta agentul ca fiind "offline".
    * **Soluție:** Aceasta s-a rezolvat prin relansarea manuală a conexiunii. Am navigat la **Manage Jenkins > Manage Nodes and Clouds > ssh-agent1** și am dat clic pe butonul **"Launch agent"**. Acest lucru a restabilit conexiunea SSH.

3.  **O cascadă de dependințe lipsă:** Aceasta a fost cea mai mare provocare. Etapa "Install Dependencies" a eșuat în mod repetat.
    * **Problema 1:** `composer: not found`.
        * **Soluție:** Am editat `Dockerfile`-ul pentru a instala `composer`.
    * **Problema 2:** `require ext-dom * -> it is missing`.
        * **Soluție:** Am editat `Dockerfile`-ul din nou pentru a instala `php-xml` și am reconstruit imaginea.
    * **Problema 3:** `require ext-mbstring * -> it is missing`.
        * **Soluție:** Am editat `Dockerfile`-ul din nou pentru a instala `php-mbstring` și am reconstruit.
    * **Problema 4:** `The zip extension and unzip/7z commands are both missing`.
        * **Soluție:** Am editat `Dockerfile`-ul pentru ultima dată pentru a adăuga `unzip` și `php-zip` și am reconstruit.

    Întregul acest proces a fost un exemplu perfect al scopului CI: a identificat sistematic toate dependențele de mediu lipsă, una câte una. Soluția a fost să tratăm `Dockerfile`-ul ca sursă unică a adevărului (single source of truth) pentru mediul de build, actualizându-l până când pipeline-ul a trecut cu succes.

# Concluzie:

În acest laborator, am configurat cu succes un pipeline CI/CD folosind Jenkins și Docker. Am definit infrastructura (controller, agent) cu docker-compose.yml și am construit o imagine de agent personalizată cu Dockerfile pentru a include toate dependențele PHP. Procesul iterativ de depanare a extensiilor (ex. ext-dom, php-zip) a fost crucial. Folosind un Jenkinsfile, am automatizat cu succes întregul flux—de la preluarea codului de pe Git, la composer install și rularea testelor phpunit—demonstrând eficiența DevOps în asigurarea unei testări automate și fiabile.

# Surse: 

Documentația oficială Jenkins:

  Instalare cu Docker: https://www.jenkins.io/doc/book/installing/docker/

  Sintaxa Jenkinsfile: https://www.jenkins.io/doc/book/pipeline/syntax/

  Utilizarea agenților: https://www.jenkins.io/doc/book/architecting-for-scale/

Documentația oficială Docker:

  Sintaxa docker-compose.yml: https://docs.docker.com/compose/compose-file/

  Referința Dockerfile: https://docs.docker.com/engine/reference/builder/

Documentația Composer: https://getcomposer.org/doc/

Documentația PHPUnit: https://phpunit.de/documentation.html
