# TP JADE : Exploration Coordonnée Multi-Robots avec Répartition de Zones

## 🎯 Objectif du TP

Créer un système multi-agent simulant une exploration coordonnée où :
- Un **CoordinatorAgent** divise une zone géographique en sous-zones
- Plusieurs **RobotAgent** explorent leurs zones assignées de manière autonome
- Le coordinateur synchronise la fin de l'exploration

---

## 📋 Prérequis

- **Java JDK 8+**
- **JADE 4.5+** (Java Agent DEvelopment Framework)
- Un IDE Java (Eclipse, IntelliJ IDEA, NetBeans, etc.)

---

## 🏗️ Architecture du Projet
```
projet-jade-exploration/
├── src/
│   └── agents/
│       ├── CoordinatorAgent.java    # Agent coordinateur
│       ├── RobotAgent.java          # Agent robot explorateur
│       └── MainContainer.java       # Point d'entrée du système
└── lib/
    └── jade.jar                     # Bibliothèque JADE
```

---

## 🧪 Concepts JADE Illustrés

✅ **Communication FIPA-ACL** (messages INFORM)  
✅ **Coordination supervisée** (CoordinatorAgent)  
✅ **Répartition des tâches** (zones distinctes)  
✅ **Autonomie locale** (chaque RobotAgent agit indépendamment)  
✅ **Synchronisation finale** (coordinateur arrête le système quand tout est fini)  

---

## 📝 Étapes du TP

### **Étape 1 : Créer le point d'entrée (MainContainer)**

Créez le fichier `MainContainer.java` pour démarrer le système.
```java
package agents;

import jade.core.Profile;
import jade.core.ProfileImpl;
import jade.core.Runtime;
import jade.wrapper.AgentContainer;
import jade.wrapper.AgentController;

public class MainContainer {
    public static void main(String[] args) {
        try {
            Runtime rt = Runtime.instance();
            Profile profile = new ProfileImpl();
            AgentContainer mainContainer = rt.createMainContainer(profile);

            // Lancer le coordinateur avec 5 robots
            AgentController coordinator = mainContainer.createNewAgent(
                    "Coordinator",
                    "agents.CoordinatorAgent",
                    new Object[]{"5"} // Nombre de robots
            );
            coordinator.start();

        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

**📌 Rôle :** Lance la plateforme JADE et crée le coordinateur qui créera ensuite les robots.

---

### **Étape 2 : Créer l'agent coordinateur (CoordinatorAgent)**

Créez le fichier `CoordinatorAgent.java` qui divise la zone et supervise l'exploration.
```java
package agents;

import jade.core.Agent;
import jade.core.AID;
import jade.lang.acl.ACLMessage;
import jade.wrapper.AgentController;

public class CoordinatorAgent extends Agent {

    private int numberOfRobots;
    private int completed = 0;

    @Override
    protected void setup() {
        System.out.println("📡 " + getLocalName() + " démarré.");
        
        // Récupération du nombre de robots depuis les arguments
        Object[] args = getArguments();
        if (args != null && args.length > 0) {
            numberOfRobots = Integer.parseInt(args[0].toString());
        } else {
            numberOfRobots = 3; // Valeur par défaut
        }

        // Diviser la zone (zone totale : 100x100)
        int zoneWidth = 100;
        int zonePerRobot = zoneWidth / numberOfRobots;

        // Créer et lancer les robots
        for (int i = 0; i < numberOfRobots; i++) {
            try {
                AgentController robot = getContainerController().createNewAgent(
                        "Robot" + i,
                        "agents.RobotAgent",
                        new Object[]{
                            i * zonePerRobot,           // Zone de départ
                            (i + 1) * zonePerRobot,     // Zone de fin
                            getLocalName()              // Nom du coordinateur
                        }
                );
                robot.start();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

    @Override
    protected void takeDown() {
        System.out.println("❌ " + getLocalName() + " terminé.");
    }

    // Réception des messages de fin d'exploration
    @Override
    protected void onMessage(ACLMessage msg) {
        if (msg.getPerformative() == ACLMessage.INFORM) {
            completed++;
            System.out.println("✅ " + msg.getSender().getLocalName() + " a terminé sa zone !");
            
            // Vérifier si tous les robots ont terminé
            if (completed == numberOfRobots) {
                System.out.println("🎉 Tous les robots ont terminé l'exploration !");
                doDelete(); // Arrêter le coordinateur
            }
        }
    }
}
```

**📌 Rôle :**
1. Divise la zone totale (100x100) en sous-zones égales
2. Crée dynamiquement les robots et leur assigne une zone
3. Reçoit les messages **INFORM** des robots
4. Termine quand tous les robots ont fini

---

### **Étape 3 : Créer l'agent robot (RobotAgent)**

Créez le fichier `RobotAgent.java` qui explore sa zone assignée.
```java
package agents;

import jade.core.Agent;
import jade.core.AID;
import jade.core.behaviours.Behaviour;
import jade.lang.acl.ACLMessage;

public class RobotAgent extends Agent {

    private int startZone;
    private int endZone;
    private String coordinatorName;

    @Override
    protected void setup() {
        // Récupération des arguments (zone assignée)
        Object[] args = getArguments();
        if (args != null && args.length == 3) {
            startZone = Integer.parseInt(args[0].toString());
            endZone = Integer.parseInt(args[1].toString());
            coordinatorName = args[2].toString();
        }

        System.out.println("🤖 " + getLocalName() + " couvre la zone [" + startZone + " - " + endZone + "]");

        // Comportement d'exploration
        addBehaviour(new Behaviour() {
            private boolean done = false;

            @Override
            public void action() {
                // Simulation de l'exploration
                System.out.println(getLocalName() + " explore la zone...");
                try {
                    Thread.sleep(1500); // Simulation du temps d'exploration
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                // Envoi du message INFORM au coordinateur
                ACLMessage msg = new ACLMessage(ACLMessage.INFORM);
                msg.addReceiver(new AID(coordinatorName, AID.ISLOCALNAME));
                msg.setContent("Zone [" + startZone + "-" + endZone + "] terminée");
                send(msg);
                
                done = true;
            }

            @Override
            public boolean done() {
                return done;
            }
        });
    }

    @Override
    protected void takeDown() {
        System.out.println("🛑 " + getLocalName() + " terminé.");
    }
}
```

**📌 Rôle :**
1. Reçoit une zone à explorer (startZone → endZone)
2. Simule l'exploration (attente de 1.5 secondes)
3. Envoie un message **INFORM** au coordinateur
4. Se termine automatiquement

---

## 🚀 Exécution du TP

### **1. Compiler le projet**
```bash
javac -cp jade.jar src/agents/*.java
```

### **2. Exécuter le système**
```bash
java -cp .:jade.jar agents.MainContainer
```

*(Sur Windows, remplacez `:` par `;`)*

---

## 📊 Résultat Attendu (Console)
```
📡 Coordinator démarré.
🤖 Robot0 couvre la zone [0 - 20]
🤖 Robot1 couvre la zone [20 - 40]
🤖 Robot2 couvre la zone [40 - 60]
🤖 Robot3 couvre la zone [60 - 80]
🤖 Robot4 couvre la zone [80 - 100]
Robot0 explore la zone...
Robot1 explore la zone...
Robot2 explore la zone...
Robot3 explore la zone...
Robot4 explore la zone...
✅ Robot0 a terminé sa zone !
✅ Robot1 a terminé sa zone !
✅ Robot2 a terminé sa zone !
✅ Robot3 a terminé sa zone !
✅ Robot4 a terminé sa zone !
🎉 Tous les robots ont terminé l'exploration !
❌ Coordinator terminé.
```

---

## 🔧 Version Améliorée avec CyclicBehaviour

Pour une gestion plus robuste des messages, voici une version améliorée du **CoordinatorAgent** utilisant un `CyclicBehaviour` :
```java
package agents;

import jade.core.Agent;
import jade.core.AID;
import jade.core.behaviours.CyclicBehaviour;
import jade.lang.acl.ACLMessage;
import jade.lang.acl.MessageTemplate;
import jade.wrapper.AgentController;

public class CoordinatorAgent extends Agent {

    private int numberOfRobots;
    private int completed = 0;

    @Override
    protected void setup() {
        System.out.println("📡 " + getLocalName() + " démarré.");
        
        Object[] args = getArguments();
        if (args != null && args.length > 0) {
            numberOfRobots = Integer.parseInt(args[0].toString());
        } else {
            numberOfRobots = 3;
        }

        int zoneWidth = 100;
        int zonePerRobot = zoneWidth / numberOfRobots;

        // Créer les robots
        for (int i = 0; i < numberOfRobots; i++) {
            try {
                AgentController robot = getContainerController().createNewAgent(
                        "Robot" + i,
                        "agents.RobotAgent",
                        new Object[]{i * zonePerRobot, (i + 1) * zonePerRobot, getLocalName()}
                );
                robot.start();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }

        // Comportement d'écoute des messages
        addBehaviour(new CyclicBehaviour() {
            @Override
            public void action() {
                MessageTemplate mt = MessageTemplate.MatchPerformative(ACLMessage.INFORM);
                ACLMessage msg = receive(mt);
                
                if (msg != null) {
                    completed++;
                    System.out.println("✅ " + msg.getSender().getLocalName() + " a terminé sa zone !");
                    
                    if (completed == numberOfRobots) {
                        System.out.println("🎉 Tous les robots ont terminé l'exploration !");
                        myAgent.doDelete();
                    }
                } else {
                    block(); // Attendre un message
                }
            }
        });
    }

    @Override
    protected void takeDown() {
        System.out.println("❌ " + getLocalName() + " terminé.");
    }
}
```

**📌 Avantages :**
- Écoute asynchrone des messages
- Filtrage par performative (INFORM)
- Gestion propre du blocage avec `block()`

---

## 🔍 Questions de Compréhension

1. **Comment le coordinateur divise-t-il la zone entre les robots ?**
2. **Que se passe-t-il si un robot termine avant les autres ?**
3. **Pourquoi utilise-t-on `Thread.sleep()` dans le robot ?**
4. **Modifiez le code pour que chaque robot ait une vitesse différente (temps d'exploration variable).**
5. **Ajoutez un système de détection d'obstacles (certaines zones prennent plus de temps).**

---

## 📚 Pour Aller Plus Loin

- **Éviter les chevauchements** : Implémenter une vérification des zones
- **Gestion des pannes** : Si un robot ne répond pas après X secondes
- **Visualisation graphique** : Afficher les zones sur une grille 2D
- **Protocole REQUEST/INFORM** : Le robot demande une zone au lieu de la recevoir automatiquement
- **Exploration adaptative** : Le coordinateur réassigne les zones en fonction de la progression

---

## 📖 Références

- [Documentation JADE](https://jade.tilab.com/)
- [FIPA ACL Message Structure](http://www.fipa.org/specs/fipa00061/)
- [JADE Behaviours Guide](https://jade.tilab.com/doc/programmersguide.pdf)

---

## 🎓 Exercices Pratiques

### **Exercice 1 : Zone 2D**
Modifiez le système pour gérer une grille 2D (10x10) au lieu d'une zone linéaire.

### **Exercice 2 : Priorités**
Ajoutez des priorités aux zones (certaines zones doivent être explorées en premier).

### **Exercice 3 : Protocole Contract Net**
Remplacez l'assignation directe par un appel d'offres (les robots proposent leur disponibilité).

### **Exercice 4 : Rapport détaillé**
Chaque robot envoie un rapport avec le nombre d'obstacles trouvés dans sa zone.

---

**✅ TP Terminé ! Vous maîtrisez maintenant la coordination multi-agents et la répartition de tâches avec JADE.**
