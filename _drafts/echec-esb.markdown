# Pourquoi mon projet d'ESB a échoué — et ce que j'en ai appris

Il existe des projets qui échouent malgré une bonne architecture, une motivation solide et une conviction profonde qu'il s'agit de la bonne direction. Mon tentative de mettre en place un Enterprise Service Bus en est l'exemple parfait.

Je vais être honnête : c'est l'un des échecs les plus formateurs de ma carrière.

---

## Le contexte

L'objectif était clair : remplacer une myriade de communications point à point, synchrones et non scalables entre les différentes applications de l'entreprise par une architecture événementielle centralisée. Un ESB pour orchestrer les flux, découpler les systèmes, et poser les bases d'une infrastructure capable de grandir.

Sur le papier, la solution avait du sens. Dans la réalité, l'atterrissage a été brutal.

---

## Un terrain qui n'était pas prêt

La première erreur — que je n'avais pas suffisamment mesurée au départ — a été de sous-estimer le delta entre la maturité nécessaire pour ce type de projet et celle de l'organisation à ce moment-là.

Aucun système de queue n'était en place. Des outils comme AWS SQS ou AWS Lambda étaient inconnus de la plupart des équipes. La notion même d'ESB nécessitait d'être expliquée à tous les niveaux : technique, qualité, management. J'ai multiplié les présentations, les sessions de sensibilisation, les argumentaires. C'est du temps bien investi en théorie, mais en pratique, il s'agissait surtout de combler un retard considérable avant même de poser la première ligne de code.

---

## Le silo infrastructure / engineering : un mur invisible mais solide

L'un des bloqueurs les plus sous-estimés a été le cloisonnement quasi hermétique entre les équipes infrastructure et engineering. Deux mondes qui coexistaient sans vraiment se parler, avec des priorités, un vocabulaire et des représentations du risque radicalement différents.

Mettre en place un ESB implique précisément de faire collaborer ces deux mondes. Sans cette collaboration, chaque décision technique devenait un sujet de négociation, chaque composant un terrain de friction.

---

## Un engineering concentré sur un seul projet

L'engineering de l'entreprise était à l'époque entièrement focalisé sur un projet central. Toutes les ADRs y faisaient référence, tous les efforts y étaient concentrés, toutes les solutions en étaient dérivées.

Ce contexte avait une conséquence directe et perverse : chaque nouveau service qui devait communiquer avec le système principal donnait lieu à des endpoints point à point, synchrones, et non scalables. Exactement le problème que j'essayais de résoudre. L'ESB n'était pas perçu comme une solution systémique, mais comme une complication supplémentaire sur un chemin déjà bien tracé.

---

## Les compromis qui fragilisent

Pour avancer malgré les frictions, j'ai accepté des compromis. Sur le design, sur le périmètre, sur les responsabilités. C'est souvent inévitable dans ce type de projet, mais chaque compromis a un coût : il fragmente la cohérence de la solution et crée des dettes que l'on finit par payer au moment le moins opportun.

Et puis est venu le moment où, les composants choisis et l'implémentation presque terminée, une nouvelle ambition s'est invitée : offrir la developer experience parfaite. Rendre la consommation des événements "magique". L'idée était séduisante. Elle a conduit à une complexification significative de la solution, à des problèmes de performance, et finalement à l'abandon du projet.

Les équipes ont continué à communiquer comme avant — point à point, synchrone, non scalable.

---

## Ce que j'en retiens

L'échec d'un projet ne remet pas en cause sa pertinence technique. Un ESB restait la bonne réponse au problème posé. Ce qui a manqué, c'est l'alignement entre l'ambition du projet et la réalité du terrain.

Quelques enseignements qui me semblent durables :

**Le mindset global d'une entreprise est une variable critique.** Ce qui paraît évident dans un contexte peut sembler absurde dans un autre. Avant de porter un projet structurant, il faut prendre le temps de cartographier sérieusement les représentations, les résistances et les priorités des différentes parties prenantes.

**La motivation du porteur de projet ne suffit pas.** Elle est nécessaire, mais loin d'être suffisante. Si les équipes ne sont pas prêtes, si le terreau n'est pas là, le projet finira par s'essouffler face aux obstacles du quotidien.

**Commencer par sonder, pas par construire.** La prochaine fois, je consacrerai davantage de temps en amont à évaluer la faisabilité organisationnelle autant que la faisabilité technique. La seconde est souvent plus simple à résoudre que la première.

**Les effets de bord positifs existent.** Ce projet a eu une conséquence inattendue et précieuse : il a commencé à briser le silo entre infrastructure et engineering. Pas suffisamment, pas assez vite, mais la conversation a été ouverte. C'est une fondation sur laquelle il est possible de construire.

---

Échouer sur un projet aussi ambitieux n'est pas agréable. Mais cette expérience m'a donné une lecture bien plus fine des dynamiques organisationnelles et des conditions réelles nécessaires à l'adoption de changements structurants. C'est une forme de capital qui, j'en suis convaincu, trouvera à s'employer.
