---
title: "Un peu de contexte"
date: 2022-06-11T12:04:16-05:00
tags: ["go"]
---

Le package [`context`](https://pkg.go.dev/context) de Go est un package très central et peut être
très puissant si il est utilisé correctement.
Par exemple, les package [`net/http`](https://pkg.go.dev/net/http) ou [`grpc`](https://pkg.go.dev/google.golang.org/grpc)
reposent sur lui.
D'un autre coté, ses concepts peuvent être un peu difficiles à comprendre au début et peuvent être confusants,
poussant les développeurs à l'ignorer ou mal l'utiliser.

Essayons d'explorer ces concepts.

<!--more-->

### Qu'est-ce que `context` ?

Comme son nom le suggère, `context` permet de passer un contexte à un programme ou une partie de celui-ci,
comme une fontion, un handler de requête ou une goroutine ; un contexte peut être des données arbitraires
ou définies ou une deadline.

Par exemple, un contexte peut être :

* Un temps maximum pour exécuter un appel réseau (aka "timeout" ou "dealine")
* Des valeurs spécifiques à une requête HTTP comme une authentification
* Une configuration d'application passée à un autre package

Un contexte est porté par une variable implémentant l'interface [`context.Context`](https://pkg.go.dev/context#Context),
qui, par convention ou best practice, est passée aux fonctions comme premier argument, par exemple :

{{< highlight go >}}
func myContextedFunc(ctx context.Context, foo string, bar interface{}) error {
    // ...
    return nil
}
{{< /highlight >}}

De cette manière, un développeur sait qu'une fonction qui prend un `context.Context` comme premier
argument prendra en compte et utilisera ce contexte et le passera en paramètre si elle appelle d'autres
fonctions.

### Alors, comment les utiliser ?

#### Les contextes racines

Les contextes sont presque tous dérivés d'un autre contexte, comme des poupées russes, ce qui veut
dire qu'ils sont créés en disant "Je prends ce contexte, j'ajoute ou supprime ces propriétés, et
voici le nouveau contexte qui en résulte".

Les deux exeptions à ceci sont `context.Background()` et `context.TODO()`.

Ces deux fonctions créent un nouveau contexte vide, avec aucune valeur ou deadline, et ne prennent
aucun argument :

{{< highlight go >}}
ctx := context.Background()
ctx2 := context.TODO()
{{< /highlight >}}

Dans les faits, si l'on regarde [les sources du package `context`](https://cs.opensource.google/go/go/+/refs/tags/go1.18.3:src/context/context.go;l=199),
on peut voir qu'elles retournent une valeur identique, `emptyCtx`, un contexte vide.

Alors quelle est la différence etre `context.Background()` et `context.TODO()` ? Principalement lexicale.

`context.Background()` ne devrait être appellée qu'une seule fois par programme, au plus haut niveau,
comme la fonction main() ou un package d'initialisation.
Dans l'image des poupées russes, ce serait la plus grande poupée, celle qui contient toutes les autres.
Le contexte retourné doit être envoyé à la première fonction prenant un contexte qui en créera un nouveau
dérivant de ce contexte vide :

{{< highlight go >}}
package main

func main() {
    ctx := context.Background()

    ctx = otherPackageA.Init(ctx)
    ctx = otherPackageB.Init(ctx)

    // ...
}
{{< /highlight >}}

Le contexte créé dans cette fonction de plus haut niveau va désormais "couler" le long du programme,
chaque partie héritant du contexte précedent en y ajoutant ses spécificités.

`context.TODO()` doit être utilisé lorsque l'on se dit *"Je dois utiliser un contexte, mais je ne sais
pas lequel encore.*. On peut voir cette fonction comme écrire le commentaire suivant :

{{< highlight go >}}
// @TODO : On envoit un nouveau contexte ici, il devrait être défini plus tard
package.FuncReceivingContext(context.TODO(), otherArg)
{{< /highlight >}}

`FuncReceivingContext` va recevoir un nouveau contexte, non dérivé des autres, avec aucune valeur ni deadline.

Certains linters et outils d'analyse statique comme [contextcheck](https://github.com/sylvia7788/contextcheck)
s'assurent que tous les contextes utilisés dérivent d'un seul `context.Background()` et échouent si le code
crée un autre contexte `Backgound()` quelque part. C'est ici que `context.TODO()` peut être utilisé.

#### Les contextes dérivés

Le package [`context`](https://pkg.go.dev/context) offre 4 fonctions qui créent des contextes dérivés.

##### `context.WithCancel(ctx context.Context) (context.Context, func())`

Un appel à `WithCancel` retourne un nouveau contexte derivé de celui passé en premier argument et une
fonction.

Un appel à la méthode `Done()` du nouveau contexte retourne un [channel](https://go.dev/tour/concurrency/2)
qui sera fermé quand on appelle la fonction retrounée, nommée la *cancel function*.

Une fois que le channel retourné par `Done()` est fermé, la méthode `Err()` du nouveau contexte
retourne l'erreur `context.Canceled` (dont le message est *"context canceled"*).

Ce pattern est utilisé pour indiquer à une fonction quand elle doit s'arrêter, par exemple :

{{< highlight go >}}
package main

import (
	"context"
	"fmt"
	"time"
)

func counter(ctx context.Context) {
	start := time.Now()

	// Attend que le channel retourné par Done() soit fermé
	<-ctx.Done()
	// Et on récupere le temps écoulé depuis qu'on a lancé la fonction
	fmt.Printf("Vous avez attendu %.2f seconde avant d'appeller cancel()\n", time.Since(start).Seconds())
}

func main() {
	// On crée un Cancel contexte depuis context.Background()
	ctx, cancel := context.WithCancel(context.Background())
	// On lance counter() dans une goroutine
	go counter(ctx)
	// On attend 0,3 seconde
	time.Sleep(300 * time.Millisecond)
	// Et on appelle cancel() pour stopper l'exécution de counter()
	cancel()

	// Un sleep en plus est rajouté pour laisser le temps a fmt.Printf
	time.Sleep(time.Millisecond)
}

// Vous avez attendu 0.30 seconde avant d'appeller cancel()
{{< /highlight >}}
*([essayer ce code !](https://go.dev/play/p/gKODzi_T46d))*

`WithCancel` est souvent utilisé avec [`defer()`](https://go.dev/tour/flowcontrol/12)
pour exécuter des taches en arrière plan pendant l'exécution du programme:

{{< highlight go >}}
package main

import (
	"context"
	"fmt"
	"time"
)

func runInBackground(ctx context.Context) {
    tick := time.NewTicker(60 * time.Second)
    go func() {
        for {
            select {
            case <-ctx.Done():
                // main s'est arrêté, on arrête cette tâche en arrière plan
                tick.Stop()
                return // return pour arrêter l'exécution de la goroutine (goroutine leak)
            case <- tick.C:
                // Faire quelque chose chaque minute pendant l'exécution du programme
            }
        }
    }()
}

func main() {
	ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    runInBackground(ctx)

    // reste du programme ...
}
{{< /highlight >}}

Si le contexte passé en argument à `WithCancel` est déjà un cancel contexte, le channel de  `Done()`
sera fermé quand la cancel function retournée sera appellée ou celle du parent, quelle que soit celle
qui est appellée en premier.

##### `context.WithDeadline(ctx context.Context, time.Time) (context.Context, func())`

`WithDeadline` crée un cancel contexte ([voir ci-dessus](#contextwithcancelctx-contextcontext-contextcontext-func)),
mais avec l'assurance que `Done()` sera fermé quand la deadline passée en second argument sera passée.

Si le channel de `Done()` est fermé suite à un appel à la cancel function, la méthode `Err()` m
retournera une erreur `context.Canceled`, mais si il est fermé car la deadline est dépassée, il
retournera une erreur `context.DeadlineExceeded` (dont le message est *"context deadline exceeded"*).

Ce pattern est utilisé pour donner à une fonction une deadline pour finir :

{{< highlight go >}}
package main

import (
	"context"
	"fmt"
	"time"
)

func faitUnTrucLent(ctx context.Context) {
	select {
	case <-time.After(1 * time.Second):
		fmt.Printf("j'ai pu faire un truc lent\n")
	case <-ctx.Done():
		fmt.Printf("je n'ai pas pu faire un truc lent car %s\n", ctx.Err())
	}
}

func faitUnTrucRapide(ctx context.Context) {
	select {
	case <-time.After(1 * time.Microsecond):
		fmt.Printf("j'ai pu faire un truc rapide\n")
	case <-ctx.Done():
		fmt.Printf("je n'ai pas pu faire un truc rapide car %s\n", ctx.Err())
	}
}

func main() {
	deadline := time.Now().Add(500 * time.Millisecond)
	ctx, cancel := context.WithDeadline(context.Background(), deadline)
	defer cancel()

	go faitUnTrucLent(ctx)
	go faitUnTrucRapide(ctx)

	time.Sleep(time.Second)
}

// j'ai pu faire un truc rapide
// je n'ai pas pu faire un truc lent car context deadline exceeded happened
{{< /highlight >}}
*([essayer ce code !](https://go.dev/play/p/xH_-YgtnNAg))*

Si le contexte parent (celui passé en premier argument) a déjà une deadline, le nouveau contexte sera
défini avec la deadline la plus récente des deux.

Même si la deadline est dépassée, il est toujours de la responsabilité de la fonction qui crée le
contexte avec deadline d'appeller la cancel function, car elle permet de libérer les ressources
nécessaires au fonctionnement de ce contexte.
Donc **TOUJOURS** appeller cancel() quand on crée des contextes de deadline.

##### `context.WithTimeout(ctx context.Context, time.Duration) (context.Context, func())`

`WithTimeout` fonctionne de la même façon que [`WithDeadline`](#contextwithdeadlinectx-contextcontext-timetime-contextcontext-func),
a la seule différence qu'elle prend non pas une date (time.Time) mais une durée (time.Duration), qui
sera ajoutée à l'instant présent (time.Now()) pour définir la deadline.

A la place de faire

{{< highlight go >}}
deadline := time.Now().Add(500 * time.Millisecond)
newContext, cancel := context.WithDeadline(previousContext, deadline)
defer cancel()
{{< /highlight >}}

On fait juste

{{< highlight go >}}
newContext, cancel := context.WithTimeout(previousContext, 500 * time.Millisecond)
defer cancel()
{{< /highlight >}}


##### `context.WithValue(context.Context, key, value any) context.Context`

`WithValue` est utilisée pour créer un nouveau contexte contenant une valeur. Cela peut être très
utile si c'est bien utilisé, en respectant quelques consignes.

De base, n'importe quelle valeur peut être utilisée pour `key` and `value` du moment qu'elles sont
d'un type qui est comparabale.

{{< highlight go >}}
newContext := context.WithValue(previousContext, "réponse", 42)
{{< /highlight >}}

Mais comme best practice (vraiment, respectez celui-ci), c'est important de ne jamais utiliser un type
built-in (string, number, struct ou interface connues) pour la valeur de `key`.
Pour un encore meilleur best practice, utilisez toujours un type non exporté du package qui utilise
cette valeur.

La raisons à cela est que tout le monde peut ajouter ou remplace une valeur à un contexte en
utilisant `WithValue`, cela veut dire que si vous utilisez une string pour `key` par exemple, une
collision avec n'importe quel autre package utilisant la même valeur pour arriver et remplacer la
valeur que vous avez définie.

Une implémentation typique de cette consigne est la suivante :

{{< highlight go >}}
package monpackage

type contextValueKeyType string

var contextValueKey = contextValueKeyType("mon nom de clé")

func SetContextValue(ctx context.Context, value string) context.Context {
    return context.WithValue(ctx, contextValueKey, value)
}

func GetContextValue(ctx context.Context) (string, error) {
    v, ok := context.Value(ctx, contextValueKey).(string)
    if !ok {
        return "", errors.New("la valeur n'est pas dans le contexte")
    }
    return v, nil
}

{{< /highlight >}}

Donc cette valeur pour être définie / lue par un autre package sans aucune collision possible :

{{< highlight go >}}
ctx := context.Background()

value, err := monpackage.GetContextValue(ctx)
// err = errors.New("la valeur n'est pas dans le contexte")

ctx = myownpackage.SetContextValue(ctx, "test")
value, err := myownpackage.GetContextValue(ctx)
// value = "test"

{{< /highlight >}}

Il est important de garder en tete que les valeurs de contexte ne sont **pas** faites pour passer
des paramètres à une fonction mais pour conserver une/des valeur(s) dans une contexte donné, par
exemple, un résultat d'authentification dans la gestion d'une requête HTTP ou une configuration
dans l'ensemble d'un programme.

### Des exemples

#### Passer un contexte : Récupérer une url en http avec un timeout

{{< highlight go >}}
package main

import (
	"context"
	"net/http"
	"time"
)

func main() {
    // On crée un contexte avec une deadline dans deux secondes
	ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    // cancel() est appellée après l'exécution pour libérer les ressources
	defer cancel()
    
    // on passe le contexte a la requête HTTP
	req, err := http.NewRequestWithContext(ctx, "GET", "http://www.google.com", nil)

	if err != nil {
		panic(err)
	}

	client := http.Client{}
	resp, err := client.Do(req)

	if err != nil {
		if err == context.DeadlineExceeded {
			panic("le fetch a pris plus de 2 secondes")
		}
		// une autre erreur est arrivée
		panic(err)
	}
	println(resp)
}
{{< /highlight >}}

#### Utiliser un contexte : Une fonction qui accepte une deadline

{{< highlight go >}}
package main

import (
	"context"
	"fmt"
	"time"
)

// slowFunction est une fonction qui prend 200ms s'exécuter
func slowFunction() int {
	time.Sleep(200 * time.Millisecond)
	return 42
}

// wrapWithDeadline va s'assurer que slowFunction() est exécutée avant la deadline du contexte
// si celui-ci en a une.
func wrapWithDeadline(ctx context.Context) (int, error) {
	// On lance slowFunction() dans une goroutine, le resultat sera envoyé dans un channel
	res := make(chan int)
	go func() {
		res <- slowFunction()
	}()

	// Si le channel de résultat recoit le retour de slowFunction en premier, le résultat est retourné
	// Si la deadline expire en premier, l'erreur context.DeadlineExceeded est retournée
	select {
	case <-ctx.Done():
		return 0, ctx.Err()
	case v := <-res:
		return v, nil
	}
}

func main() {
	ctx := context.Background()

	ctx1, cancel1 := context.WithTimeout(ctx, 100*time.Millisecond)
	defer cancel1()
	res, err := wrapWithDeadline(ctx1)
	fmt.Println(res) // 0
	fmt.Println(err) // context deadline exceeded

	ctx2, cancel2 := context.WithTimeout(ctx, 300*time.Millisecond)
	defer cancel2()
	res, err = wrapWithDeadline(ctx2)
	fmt.Println(res) // 42
	fmt.Println(err) // <nil>
}
{{< /highlight >}}
*([essayer ce code !](https://go.dev/play/p/omNlNVjsayK))*