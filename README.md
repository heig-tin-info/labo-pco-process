# Processus (POSIX)

- **Durée**: 2 périodes
- **Format**: Travail individuel

## Objectifs

Ce travail pratique a pour objectif de vous familiariser avec la gestion des processus sous Linux, en explorant leur création, leurs états, leur gestion et leurs interactions avec le système.

À l'issue de ce TP, vous serez capable de :

- Comprendre ce qu'est un processus et ses attributs.
- Manipuler les processus avec `ps`, `top`, `htop` et `/proc`.
- Distinguer processus, démons et TTY.
- Comprendre et manipuler `fork()`, `execve()`, et les statuts des processus.
- Observer et manipuler des processus zombies et orphelins.
- Utiliser `strace` pour intercepter les appels système.
- ... et bien plus encore.

## Introduction aux Processus

### Définition et attributs

Un **processus** est une *instance* d'un programme en cours d'exécution. Chaque processus est identifié par un **PID (Process ID)** et possède divers attributs tels que:

- **UID/GID** : Propriétaire du processus.
- **PPID** : Identifiant du parent (*Parent Process ID*).
- **Statut** : Exécuté, suspendu, en attente, zombie...
- **TTY** : Terminal associé.
- **Consommation de ressources** : CPU, mémoire, etc.
- **Priorité** : Niveau d'ordonnancement.
- ...

En outre, un processus possède un **espace d'adressage** qui contient le code, les données, la pile, les variables, etc. Chaque processus a donc son propre espace d'adressage isolé des autres processus. Cela signifie que les variables locales d'un processus ne sont pas accessibles par un autre processus.

Un processus n'a en soi pas beaucoup de pouvoir. Il ne peut pas accéder directement au matériel, il ne peut pas lire ou écrire directement dans un fichier, il ne peut pas allouer de la mémoire, etc. Pour effectuer ces opérations, le processus doit demander au noyau de le faire pour lui. C'est le rôle des **appels système**.

### Cycle de vie

Un processus suit un cycle de vie qui comprend plusieurs étapes :

1. **Création** : Un processus est créé par un autre processus (son parent) via un appel système `fork()`. Il est admis dans la file d'attente des processus prêts à être exécuté.
2. **Prêt** : Le processus est prêt à être exécuté par le planificateur CPU.
3. **Actif** : Une fois les ressources allouées, le processus est exécuté par le CPU jusqu'à ce qu'il se termine, qu'il soit mis en attente ou qu'il ait épuisé son quantum de temps.
4. **Attente** : Le processus est mis en attente lorsqu'il attend une ressource (p. ex. un fichier, un signal, etc.).
5. **Zombie** : Le processus a terminé son exécution mais n'a pas encore été récupéré par son parent. Il est en attente de récupération.
6. **Destruction** : Le processus est retiré de la liste des processus actifs et ses ressources sont libérées.

Durant un même cycle de vie, un processus peut passer par plusieurs états. Par exemple, un processus peut être créé, devenir actif, être mis en attente, reprendre son exécution, etc.

### Conventions et ressources initiales

La norme POSIX (Portable Operating System Interface) définit un certain nombre de conventions pour les processus.

- **STDIN** : Une entrée standard pour les données est généralement automatiquement allouée sur le descripteur de fichier 0.
- **STDOUT** : Une sortie standard pour les données est généralement automatiquement allouée sur le descripteur de fichier 1.
- **STDERR** : Une sortie standard pour les erreurs est généralement automatiquement allouée sur le descripteur de fichier 2.
- Une fois les ressources allouées et les bibliothèques dynamiques chargées, la fonction `main()` est appelée en transmettant les arguments d'entrées du programme.
- La fonction `main()` retourne un entier qui est généralement 0 pour indiquer une sortie normale. Cet entier est retourné au processus parent.

## Exploration des processus

### Le système de fichier `/proc`

Le noyau (*kernel*) expose les informations de ses processus via plusieurs interfaces, notamment `/proc` qui contient des informations sur les processus en cours d'exécution. Il s'agit d'un système de fichier virtuel géré par le noyau qui fournit des informations sur les processus, les ressources système, les périphériques, etc.

Essayez de lister le contenu de `/proc` avec `ls /proc`. Vous constaterez que chaque sous-répertoire correspond à un processus en cours d'exécution, nommé par son **PID**. Il y a des nombres très petit et très grand, ce que vous constatez c'est notamment:

1. La présence du PID 1 qui est le processus `init`.
2. Un certain nombre de PID < 1000 qui sont des processus système.
3. Des valeurs de PID plus élevées qui sont des processus utilisateur.

Votre terminal (bash) dispose d'une variable nommée `$$` qui contient le PID du processus courant. Vous pouvez l'afficher avec `echo $$`. Vous pouvez également afficher le PID du dernier processus exécuté avec `$!` ou le PID du processus parent avec `$PPID`. Essayez de les afficher.

Essayez également de lister le statut de votre terminal avec :

```sh
cat /proc/$$/status
```

Identifier le processus parent (p. ex. `PPid:   3663246`) et afficher son statut jusqu'à remonter au processus `init` (PID 1).

```sh
cat /proc/$(cat /proc/$$/status | grep -Po 'PPid.*?\K[0-9]+')/status # ...
```

Je vous l'accorde, cette méthode n'est pas la plus pratique pour explorer les processus. Heureusement, Linux fournit des commandes pour afficher et gérer les processus, notamment `ps`, `top`, `htop`, `pgrep`, `pkill`, etc.

### Commande `ps`

La commande `ps` permet d'afficher les processus **actifs**. Elle se base bien entendu sur le système de fichier `/proc`.

Essayez de l'exécuter sans argument. Vous pouvez également utiliser `ps aux` pour afficher tous les processus de tous les utilisateurs.

```sh
ps aux
```

Explication :

- `a` : Affiche tous les processus appartenant à tous les utilisateurs.
- `u` : Format utilisateur (affiche UID, CPU, mémoire, etc.).
- `x` : Inclut les processus sans terminal associé.

Intéressez-vous à la colonne `STAT` qui indique le statut du processus :

- `S` : dormant mais interruptible
- `s` : meneur de session
- `Z` : zombie
- `T` : arrêté
- `t` : arrêté par le débogueur
- `D` : non interruptible (en attente de ressources)
- `<` : processus de haute priorité
- `N` : processus de basse priorité
- `+` : processus en avant-plan
- ...

Vous pouvez avoir la liste complète avec `man ps` et en recherchant `PROCESS STATE CODES` avec `/`.

Testons ces différents statuts... Lancez un processus en arrière-plan avec `sleep 60 &` puis affichez les processus actifs avec `ps aux`. Vous devriez voir le processus `sleep` en arrière-plan avec un statut `S`.

```sh
ps axu | grep sleep
```

Refaites la même chose sans le `&` et cette fois-ci pendant que le processus tourne, envoyez un signal `SIGSTP` (Ctrl+Z) pour le mettre en pause. Vous devriez voir le statut du processus changer en `T` (arrêté) :

```sh
ps axu | grep sleep
```

### Commande `top`

`ps` est très pratique mais il n'est pas dynamique. `top` est une commande interactive qui affiche les processus en cours d'exécution en temps réel. Elle est très utile pour surveiller les processus, les ressources système, etc.

Vous pouvez utiliser `h` pour afficher l'aide. Jouez un peu avec les fonctionnalités (couleur, forest...).

### Commande `htop`

`htop` est une version améliorée de `top`. Elle est plus conviviale, plus interactive et plus riche en fonctionnalités. Vous pouvez l'installer avec `sudo apt install htop`.

## Création de Processus

### `fork()`: Duplication de processus

Dans les premiers systèmes UNIX (notamment Version 6 d'Unix en 1975), il n'existait aucun appel système permettant de créer un processus « vierge » directement. Le modèle adopté était celui de la création de processus par duplication d'un processus existant. C'est le rôle de l'appel système `fork()` (diviser).

Lorsqu'un processus invoque `fork()`, le noyau crée une **copie exacte** du processus appelant, appelée **processus enfant**. Le processus enfant hérite de tous les attributs du processus parent, y compris le code, les données, les descripteurs de fichiers, etc. Le processus enfant est identifié par un PID unique et possède un PPID qui est le PID du processus parent.

Imaginez que vous avez une liste de tâches à effectuer (votre listing de programme) et qu'à un moment vous rencontrez l'ordre "supercalifrastilistiquexpialidocious". C'était peut-être magique, mais vous n'avez rien remarqué. Et pourtant...

À ce moment précis, vous avez été dupliqué. Le monde que vous avez connu est maintenant partagé entre vous et votre clone. Votre clone ne sait même pas qu'il est un clone. Il a lu l'ordre comme vous, il a continué à lire la liste comme vous. Il a même continué à exécuter les tâches comme vous. Mais il est un clone.

Sauf que, l'un est le processus originel, l'autre est le processus enfant, et il y a un moyen de les distinguer. C'est la valeur de retour de `fork()`. Dans le processus enfant, `fork()` retourne 0. Dans le processus parent, `fork()` retourne le PID du processus enfant.

```c
#include <stdio.h>
#include <unistd.h>

int main() {
    pid_t pid = fork();
    if (pid < 0) { perror("fork"); exit(1); }

    if (pid == 0) {
        printf("I am the child, my PID is %d, my parent is %d\n",
            getpid(), getppid());
    } else {
        printf("I am the parent, my PID is %d, my child is %d\n",
            getpid(), pid);
    }
}
```

Essayez d'exécuter ce programme. Vous devriez voir deux lignes de sortie, une pour le processus parent et une pour le processus enfant.

### `wait()`: Attente de la fin d'un processus

Dans la philosophie Unix, un processus parent à le devoir, la responsabilité de récupérer le statut de sortie de ses enfants lorsqu'ils décèdent...

C'est le rôle de l'appel système `wait()`. Lorsqu'un processus invoque `wait()`, il attend que l'un de ses enfants se termine. Si un enfant est déjà terminé, `wait()` retourne immédiatement. Sinon, le processus parent est mis en attente jusqu'à ce qu'un enfant se termine.

Dans le cas ou le processus parent à plusieurs enfants, il peut attendre tous ces enfants avec `wait(NULL)` ou un enfant spécifique avec `waitpid(pid, &status, options)`.

Imaginons un processus qui invoque 10 enfants. Chaque enfant attend une valeur aléatoire entre 10 et 20 secondes avant de se terminer. Le processus parent attend la fin de chaque enfant et affiche son PID et son statut de sortie.

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <time.h>

int main() {
    srand(time(NULL)); // Initialise l’aléatoire

    for (int i = 0; i < 10; i++) {
        pid_t pid = fork();
        if (pid < 0) { perror("fork"); exit(1); }
        if (pid == 0) {
            srand(getpid());
            int wait_time = rand() % 10 + 10; // Aléatoire entre 10 et 20 secondes
            printf("Child %d (PID %d) sleeping for %d seconds\n", i, getpid(), wait_time);
            sleep(wait_time);
            printf("Child %d (PID %d) exiting\n", i, getpid());
            exit(0);
        }
    }

    int status;
    for (int i = 0; i < 10; i++) {
        pid_t pid = wait(&status);
        printf("Parent: Child (PID %d) exited with status %d\n", pid, WEXITSTATUS(status));
    }
}
```

Pour tester, je vous propose d'utiliser l'outil `htop` qui est une version améliorée de `top` laquelle est une version améliorée de `ps`. `htop` est un outil interactif qui permet de visualiser les processus en cours d'exécution.

Ouvrez un second terminal. Une fois lancé `htop`, vous pouvez voir les processus en cours d'exécution, les processus zombies, les processus en attente, etc. Vous pouvez également tuer des processus, changer leur priorité, etc. On va filtrer et afficher les processus sous forme d'arbre.

1. Lancez `htop`
2. Appuyez sur `F4` pour afficher les processus sous forme d'arbre.
3. Filtrer `children` avec `F4` et `children` puis `Enter`.
4. Vous ne devriez voir plus rien.

Ensuite, compiler votre programme avec `gcc children.c -ochildren` et exécutez-le.

Pourquoi est-ce que la ligne `srand(getpid());` est très importante ?

### `exec()`: Exécution de programmes

Lorsque vous exécutez une commande dans un terminal (p.ex `ls -l`), le shell crée un processus pour exécuter la commande. C'est le processus **fils** (*child process*). Le processus **parent** (*shell*) attend que le processus fils se termine. C'est le modèle de création de processus que nous avons vu précédemment. Donc cela implique un `fork()`. Mais comment le processus fils exécute-t-il la commande ?

C'est le rôle de la famille d'appels système `exec`. L'appel système `exec` remplace l'image mémoire du processus courant par un nouveau programme.

```c
#include <stdio.h>
#include <unistd.h>

int main() {
    char *args[] = {"/bin/ls", "-l", NULL};
    execve(args[0], args, NULL);
    perror("execve a échoué");
    return 1;
}
```

Voici un programme qui demande à l'utilisateur de saisir du texte et transmet cette commande à `cowsay`.

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/wait.h>

int main() {
    while (!feof(stdin)) {
        printf("> ");
        char cmd[256];
        fgets(cmd, sizeof(cmd), stdin);
        pid_t pid = fork();
        if (pid < 0) { perror("fork"); return 1; }
        if (pid == 0) {
            execlp("cowsay", "cowsay", cmd, NULL);
            return 1;
        }
        wait(NULL);
    }
}
```

Comprenez-vous pourquoi le `fork()` est nécessaire quand même ?

## Appels systèmes

Nous l'avons évoqué plus haut, un processus est très limité dans ce qu'il peut faire. Il ne peut pas accéder directement au matériel, il ne sait pas lire ou écrire directement dans un fichier, il ne peut pas allouer de la mémoire, etc. Pour effectuer ces opérations, le processus doit demander au noyau de le faire pour lui. C'est le rôle des **appels système**.

À chaque appel système, le système d'exploitation est interrompu et passe en mode noyau. Le noyau effectue l'opération demandée et retourne le contrôle au processus.

Les appels système sont des fonctions de très bas niveau qui permettent aux processus d'interagir avec le noyau. Ils sont même tellement fondamentaux que l'architecture x86 dispose de l'instruction `syscall` pour effectuer un appel système de manière plus optimale qu'une interruption logicielle.

Le programme `strace` est très utile pour observer les appels système effectués par un programme. Essayez de l'exécuter sur les différents programmes que nous avons vus.

```sh
strace ./children
strace ./cowsay
...
```

Intéressons-nous à l'aide de `syscall` (`man  syscall`). Naviguez jusqu'à *Architecture calling conventions*. Vous noterez que sous x86-64 (si vous êtes sous PC), le numéro d'appel système est passé via le registre `rax` du processeur.

Allez maintenant dans le manuel `man syscalls` vous aurez la liste de tous les appels système disponibles.

### Ptrace

`ptrace` est une fonction système qui permet à un processus parent de contrôler l'exécution d'un processus fils. C'est un outil puissant pour le débogage et la surveillance des processus.

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/ptrace.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <sys/user.h>
#include <unistd.h>

int main() {
    pid_t child = fork();

    if (child == 0) {
        // Activer le traçage pour l'enfant
        ptrace(PTRACE_TRACEME, 0, NULL, NULL);
        execl("/bin/ls", "ls", NULL); // Exécuter un programme cible
        exit(0);
    }

    while (1) {
        int status;
        waitpid(child, &status, 0);
        if (WIFEXITED(status)) break; // Sortir si le processus se termine

        // Récupérer les registres du processeur
        if (WIFSTOPPED(status) && WSTOPSIG(status) == SIGTRAP) {
            struct user_regs_struct regs;
            ptrace(PTRACE_GETREGS, child, NULL, &regs);
            printf("Syscall number: %lld\n", regs.orig_rax);
        }

        ptrace(PTRACE_SYSCALL, child, NULL, NULL); // Continuer l'exécution
    }
}
```

## Signaux et Gestion des Processus

Un **signal** est un message asynchrone envoyé à un processus pour l'informer d'un événement. Les signaux sont utilisés pour la communication entre processus et pour la gestion des processus. Les signaux ne véhiculent pas de données, ils sont simplement des notifications.

L'utilitaire `kill` permet d'envoyer des signaux à un processus. Par exemple, pour tuer un processus, on envoie le signal `SIGKILL` (9) :

```sh
kill -9 <PID>
```

On peut donc envoyer des signaux à un processus pour lui demander de s'arrêter, de se mettre en pause, de reprendre, etc :

```sh
kill -SIGSTOP <PID>   # Mettre en pause
kill -SIGCONT <PID>   # Reprendre
kill -SIGKILL <PID>   # Forcer l'arrêt
```

Les signaux les plus utilisés sont généralement `SIGINT` (CTRL+C) pour interrompre un processus `SIGSTP` (CTRL+Z) pour mettre en pause un processus et `SIGKILL` pour forcer l'arrêt. Ces signaux peuvent être facilement interceptés et gérés par un processus.

La gestion de signaux était gérée à l'origine par la fonction `signal()`. Cependant, cette fonction est obsolète et a été remplacée par `sigaction()`.

```c
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <unistd.h>

void handler(int signum) {
    printf("\nVOUS NE QUITTEREZ... PAS !!!!\n");
    // exit(0);
}

int main() {
    struct sigaction sa = { .sa_handler = handler, .sa_flags = 0 };
    sigemptyset(&sa.sa_mask); // Bloque tous les signaux pendant le traitement

    if (sigaction(SIGINT, &sa, NULL) == -1) {
        perror("sigaction");
        exit(EXIT_FAILURE);
    }

    while (1) {
        printf("Zzzzzzz\n");
        pause();
        printf("Tient, je suis réveillé !\n");
    }
}
```

Notez le comportement du programme lorsqu'il reçoit un signal `SIGINT` (CTRL+C), et l'effet de l'appel système `pause()`.

Comment quitter le programme ? Une suggestion et de faire CTRL+Z, et kill le processus avec `kill <pid>`, lequel PID est indiqué lors de la suspension du processus.

## Zombies et Orphelins

### Zombie

Allons plus loin. On va s'intéresser au petit programme suivant :

```sh
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/wait.h>

int main(void)
{
    pid_t pid = fork();
    if (pid == 0) exit(0); // Child dies here
    sleep(100); // Parent sleeps for 100 seconds
    int status;
    wait(&status);
}
```

Compilez-le et exécutez-le. Que voyez-vous dans `ps axu` ? Vous devriez voir un processus zombie. Pourquoi ?

### Orphelin

Un **processus orphelin** est un processus dont le processus parent s'est terminé avant lui. Lorsqu'un processus parent se termine avant ses enfants, les enfants sont adoptés par le processus `init` (PID 1). `init` est le premier processus lancé par le noyau et est le parent de tous les processus.

## Démon

Un **démon** est un processus qui s'exécute en arrière-plan, généralement sans terminal attaché (TTY). C'est-à-dire que l'on ne peut pas interagir directement avec lui. Les démons sont souvent utilisés pour des tâches de maintenance, de surveillance, de journalisation, etc. Une base de données, un serveur web, un service de messagerie, etc., sont des exemples de démons.

En règle général lorsqu'ils doivent communiquer avec l'utilisateur, ils le font via des fichiers de log. Profitons de l'occasion pour introduire une nouvelle commande : `pstree`. Elle permet d'afficher les processus sous forme d'arbre simplement. Testez là et observez...

Vous devriez voir plusieurs processus `systemd` qui sont des démons. Systemd est un gestionnaire de système et de services pour Linux. Il est responsable du démarrage, de l'arrêt et de la gestion des services système.

Dans cet exemple on voit que `systemd` est le processus parent de services tels que `gpg-agent` ou `journal`.

```text
    ...
    ├─systemd─┬─(sd-pam)
    │         ├─dbus-daemon
    │         ├─gpg-agent───{gpg-agent}
    │         ├─2*[pipewire───2*[{pipewire}]]
    │         ├─pipewire-pulse───2*[{pipewire-pulse}]
    │         └─wireplumber───5*[{wireplumber}]
    ├─systemd-journal
    ├─systemd-logind
    ...
```

L'intérêt de ce processus intermédiaire est de permettre la surveillance et le redémarrage des services en cas de problème. C'est un processus qui a un rôle de supervision. Il veille à ce que le service tourne, il informe le système en cas de problème, il peut redémarrer le service en cas de besoin.

Il est fréquent, lorsque l'on crée un démon d'utiliser la commande `setsid` pour détacher le processus de son terminal. Cela permet de le rendre indépendant du terminal et de le rendre autonome, car si le terminal est fermé, les processus qui ont été invoqués depuis ce dernier sont tués.

Voici un exemple de tueur d'anges:

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/stat.h>

void daemonize() {
    pid_t pid = fork();
    if (pid < 0) exit(EXIT_FAILURE);
    if (pid > 0) exit(EXIT_SUCCESS); // Parent exits

    // Child becomes the session leader
    if (setsid() < 0) exit(EXIT_FAILURE);

    // Redirect standard files to /dev/null (no output)
    int fd = open("/dev/null", O_RDWR);
    dup2(fd, STDIN_FILENO);
    dup2(fd, STDOUT_FILENO);
    dup2(fd, STDERR_FILENO);
    close(fd);

    // Kill angels every 5 seconds
    while (1) {
        sleep(5);
        system("pkill angel");
    }
}

int main() {
    daemonize();
}
```

Compilez ce programme et exécutez-le. Vous devriez voir un processus `daemon` qui tourne en arrière-plan. Pour le tuer, vous pouvez utiliser `pkill daemon`.

Essayez de faire tourner des processus `angel` en arrière-plan et voir s'ils sont tués toutes les 5 secondes.



La convention ABI (Application Binary Interface) utilisée sous Linux est que les appels système sont effectués via le registre processeur `rax`. L'injection de `ptrace` force la génération du signal `SIGTRAP` lorsqu'un appel système est effectué. C'est le moment idéal pour inspecter les registres du processeur.

## Priorité d'exécution

Chaque processus a une priorité d'exécution. Les priorités vont de -20 (la plus haute) à 19 (la plus basse). La priorité par défaut est de 0. Un processus avec une priorité plus élevée sera exécuté avant un processus avec une priorité plus basse.

La commande `nice` permet de démarrer un processus avec un *niceness* (facteur de courtoisie) spécifique. Par exemple, pour démarrer un processus avec une priorité de 10 :

```sh
nice -n 10 ./myprogram
```

La commande `renice` permet de modifier la priorité d'un processus existant. Par exemple, pour augmenter la priorité d'un processus avec PID 1234 :

```sh
renice -n -5 -p 1234
```

Pourquoi est-ce utile ? La priorité des processus impacte l'attribution du temps CPU. Plus un processus a une priorité élevée, plus l'ordonnanceur lui allouera de temps CPU par rapport à d'autres processus. Les tâches essentielles qui nécessitent une réponse rapide, ou qui ont une interaction avec l'utilisateur (p. ex. un serveur web) peuvent bénéficier d'une priorité plus élevée. Le cas extrême sont les processus avec priorité `-20` qui sont dit *temps réel*.

## Quiz

1. Qu'est-ce qu'un processus ?
   - [ ] Une instance d’un programme en exécution
   - [ ] Une fonction système du noyau
   - [ ] Un fichier binaire
   - [ ] Une tâche du planificateur CPU

2. Comment obtenir le PID d’un processus en cours d’exécution ?
   - [ ] `pidof <name>`
   - [ ] `ls /proc/<name>`
   - [ ] `ps aux | grep <name>`

3. Quel est le rôle de `fork()` ?
   - [ ] Créer un nouveau thread
   - [ ] Cloner un processus en dupliquant son espace mémoire
   - [ ] Remplacer l’image mémoire d’un processus
   - [ ] Terminer un processus

4. Qu'est-ce qu'un processus zombie ?
   - [ ] Un processus en attente de terminaison
   - [ ] Un processus qui consomme trop de mémoire
   - [ ] Un processus qui n'a pas été correctement récupéré par son parent
   - [ ] Un processus en exécution perpétuelle

5. Quel est le signal envoyé par défaut par la commande `kill` ?
   - [ ] `SIGKILL`
   - [ ] `SIGSTOP`
   - [ ] `SIGTERM`
   - [ ] `SIGHUP`

6. Que fait la commande `kill -STOP <PID>` ?
   - [ ] Tue immédiatement le processus
   - [ ] Met le processus en arrière-plan
   - [ ] Suspend le processus
   - [ ] Redémarre le processus

7. Quelle est la principale différence entre `fork()` et `execve()` ?
   - [ ] `fork()` clone un processus, `execve()` le remplace
   - [ ] `fork()` termine un processus, `execve()` le met en veille
   - [ ] `fork()` crée un démon, `execve()` modifie son UID
   - [ ] `execve()` est un alias de `fork()`

8. Comment récupérer le statut de sortie d’un processus fils ?
   - [ ] `getpid()`
   - [ ] `exit()`
   - [ ] `waitpid()`
   - [ ] `kill -9 <PID>`

9. Pourquoi un processus peut-il devenir orphelin ?
   - [ ] Il n’a plus de mémoire allouée
   - [ ] Son parent est terminé avant lui
   - [ ] Il n’a pas de fichier de sortie standard
   - [ ] Il est bloqué sur une ressource système

10. Comment afficher les appels système d'un processus en cours d'exécution ?
    - [ ] `htop`
    - [ ] `ptrace`
    - [ ] `strace`
    - [ ] `ps aux`

11. Que fait `setsid()` ?
    - [ ] Détache un processus de son terminal
    - [ ] Bloque un processus
    - [ ] Crée un thread
    - [ ] Termine un processus

12. Comment un processus parent récupère-t-il la sortie d’un processus fils terminé ?
    - [ ] Avec `wait()`
    - [ ] Avec `execve()`
    - [ ] Avec `kill -9`
    - [ ] Avec `ps aux`

13. Que se passe-t-il si un processus ignore `SIGKILL` ?
    - [ ] Il est détruit par le noyau
    - [ ] Il continue de s’exécuter
    - [ ] Il devient un démon
    - [ ] Ce n’est pas possible, `SIGKILL` ne peut pas être ignoré

14. Quelle commande peut modifier la priorité d’un processus existant ?
    - [ ] `nice`
    - [ ] `renice`
    - [ ] `kill`
    - [ ] `fork`

15. Qu’est-ce qu’un namespace en Linux ?
    - [ ] Une isolation des ressources pour les processus
    - [ ] Un type de permission utilisateur
    - [ ] Un identifiant de fichier
    - [ ] Un PID système
