---
layout: post
title: Les diamants sont éternels
subtitle: Une tentative sur le kata Diamond
tags: [php, TDD, kata]
---

Au hasard de mes rebonds sur Internet, je suis tombé sur une [publication récente](https://blog.crafting-labs.fr/2018/01/29/diamond-kata/) à propos du kata Diamond. Comme je m'intéresse au TDD ces derniers temps, j'ai voulu tenté ma chance moi aussi et voir quel design allait en émerger.

Le sujet est très simple à présenter ; je reprend la [définition de Seb Rose](http://claysnow.co.uk/recycling-tests-in-tdd/) qui semble être la référence d'origine de ce problème :
>Given a letter, print a diamond starting with ‘A’ with the supplied letter at the widest point.

Par exemple, le diamant 'C' correspond à : 
```
  A  
 B B 
C   C
 B B 
  A  
``` 

Je suis en train de prendre en main PHPspec, outil destiné à faire du BDD. C'était aussi l'occasion de le mettre en pratique sur un exemple simple. Je reviendrai sur l'outil dans un prochain billet, il n'est ici question que d'enchaîner des tests qui échouent, d'écrire des bouts de code pour les faire passer, et de réorganiser tout ça.

# Test #1 : Partir de quelque part

Mon premier test est de s'assurer d'être capable de construire le diamant 'A' :
```
A
```
Il faut bien partir de quelque part. C'est le premier diamant et en ce sens un cas limite qu'il faut de toute façon prendre en compte dans les tests à un moment ou un autre. Autant le faire d'entrée de jeu.

Cela donne le premier test :
```php
public function it_should_build_diamond_a() {
    $this->create('A')->shouldReturn('A');
}
```
Evidemment, ce test échoue, la méthode `create` (générée automatiquement par PHPSpec) retournant pour l'instant `null`.

# Code #1 : Bête et méchant

Le code minimum pour faire passer le test est simplement de changer la valeur de retour en `A` :

```php
public function create($letter) {
    return 'A';
}
```
C'est suffisant pour valider cette étape. Le code actuel ne pouvant pas vraiment être refactorisé, ajoutons un second test.

# Test #2 : le diamant 'B'

Le diamant 'A' ne donne pas beaucoup d'information sur le code à produire, mais il est important car c'est un cas limite où le diamant se limite à un seul élément. Sans trop savoir où aller, je poursuit avec le cas du diamant 'B' en ajoutant à ma spécification :
```php
public function it_should_build_diamond_b() {
    $this->create('B')->shouldReturn(" A \nB B\n A ");
}
```
Ce test ne passe évidemment pas.

# Code #2 : Encore bête et encore méchant

Le code le plus simple pour gérer ce nouveau cas d'utilisation est de tester l'argument pour savoir si c'est le diamant 'A' ou le diamant 'B' qui doit être retourné.

```php
public function create($letter) {
    if ('A' === $letter) {
        return 'A';
    }
    return " A \nB B\n A ";
}
```
# Refacto #1 : Convenance personnelle

Je n'aime pas du tout les chaînes de caractères comme " A \nB B\n A " à cause des caractères de contrôle. Je trouve beaucoup plus pratique de les représenter comme un tableau de lignes que je concatène quand j'ai besoin de la chaîne entière. Cela permet d'écrire la chaîne en respectant sa mise en forme réelle. J'ai donc procédé à la réorganisation du code suivante :
```php
public function create($letter) {
    if ('A' === $letter) {
        return 'A';
    }
    $diamond = [
        ' A ',
        'B B',
        ' A ',
    ];
    return implode("\n", $diamond);
}
```
Les tests passent toujours...

# Refacto #2 : Le jeu des 7 points communs
L'objectif étant de trouver une généralisation à tous les cas possibles, il m'a semblé nécessaire de faire apparaître les similitudes de construction et donc de traiter le diamant A et le diamant B de la manière : test sur `$letter` pour savoir quel diamant générer, représentation du diamant sous la forme d'un tableau de ligne et retour de la chaîne résultante :
```php
public function create($letter) {
    if ('A' === $letter) {
        $diamond = [
            'A',
        ];
    }
    if ('B' === $letter) {
      $diamond = [
          ' A ',
          'B B',
          ' A ',
      ];
    }
    return implode("\n", $diamond);
}
```
Les tests passent toujours...

# Refacto #3 : Miroir, mon 'B' miroir...
Maintenant que la forme du diamant 'B' apparaît dans le code, ce qui me saute aux yeux, c'est sa symétrie horizontale. La ligne du milieu sert de pivot, ce qui ne me paraît pas le plus pratique pour profiter de la symétrie. Je fais apparaître artificellement une nouvelle ligne dans le diamant :
```php
$diamond = [
    ' A ',
    'B B',
//  'B B',
    ' A ',
];
```
La connaissance du haut du diamant est suffisante pour le construire entièrement : les lignes du bas sont les lignes du haut prises dans l'ordre inverse, en excluant la première. Cela donne :

```php
$upper = [
    ' A ',
    'B B',
];
$lower = array_reverse($upper);
$lower = array_slice($lower, 1);
$diamond = array_merge($upper, $lower);
```
Les tests passent toujours...

# Refacto #4 : A-symétrie
Finalement, le diamant 'A' présente la même symétrie :
```php
$diamond = [
    ' A ',
//  ' A ',
];
```
Alors, autant mutualiser le code :
```php
public function create($letter) {
    if ('A' === $letter) {
        $upper = [
            'A',
        ];
    }
    if ('B' === $letter) {
      $upper = [
          ' A ',
          'B B',
      ];
    }
    $diamond = array_merge(
        $upper,
        array_slice(array_reverse($upper), 1)
    );
    return implode("\n", $diamond);
}
```
Les tests passent toujours...

# Test #3 : 'C'-ullinan

J'ai besoin d'un peu plus d'exemple pour aller plus loin. Je rajoute donc un test sur un plus gros diamant, le diamant 'C' :
```php
public function it_should_build_diamond_c() {
    $this->create('C')->shouldReturn("  A  \n B B \nC   C\n B B \n  A  ");
}
```
Mon code ne passe pas ce test.

# Code #4 : Toujours bête et toujours méchant

Encore une fois, le plus simple pour passer le nouveau test est de définir un traitement spécifique :
```php
public function create($letter) {
    if ('A' === $letter) {
        $upper = [
            'A',
        ];
    }
    if ('B' === $letter) {
      $upper = [
          ' A ',
          'B B',
      ];
    }
    if ('C' === $letter) {
      $upper = [
          '  A  ',
          ' B B ',
          'C   C',
      ];
    }
    $diamond = array_merge(
        $upper,
        array_slice(array_reverse($upper), 1)
    );
    return implode("\n", $diamond);
}
```
Les tests passent de nouveau.

# Refacto #5 : Une symétrie peut en cacher une autre

La symétrie horizontale m'avait sauté aux yeux avec le diamant 'B', mais il était déjà possible de voir aussi une symétrie verticale. Il m'a fallu un test (exemple) de plus pour m'en rendre compte. Sur le haut du diamant 'D' :
```php
$upper = [
   '  A' . /*A*/'  ',
   ' B ' . /* */'B ',
   'C  ' . /* */' C',
];
```
La connaissance de la gauche du diamant est suffisante pour le construire entièrement : les colonnes de droite sont les colonnes de gauche prises dans l'ordre inverse, en excluant la première.
```php
$upperLeft = [
    '  A',
    ' B ',
    'C  '
];
$upper = array_map(
    function ($left) {
        $right = strrev($left);
        $right = substr($right, 1);
        return $left . $right;
    },
    $upperLeft
);
```
Les tests passent toujours après cette refactorisation du cas du diamant 'C'...

# Refacto #6 : A- et B-symétries

Les diamants 'A' et 'B' présentent la même symétrie verticale et le code peut être mutualisé.

```php
public function create($letter) {
    if ('A' === $letter) {
        $upperLeft = [
            'A',
        ];
    }
    if ('B' === $letter) {
        $upperLeft = [
            ' A',
            'B ',
        ];
    }
    if ('C' === $letter) {
        $upperLeft = [
            '  A',
            ' B ',
            'C  ',
        ];
    }
    $upper = array_map(
        function ($left) {
            return $left . substr(strrev($left), 1);
        },
        $upperLeft
    );
    $diamond = array_merge(
        $upper,
        array_slice(array_reverse($upper), 1)
    );
    return implode("\n", $diamond);
}
```

# Refacto #7 : La diagonale des lettres

Les refactorisations précédentes ont mis en évidence que le haut gauche du diamant est un carré d'espaces dont la diagonale est constituée des lettres de 'A' à celle du diamant. Pour l'exemple du diamant 'C', par exemple, on peut expliciter les suffixes et les préfixes en espaces de chaque ligne :
```php
$upperLeft = [
    str_repeat(' ', 2) . 'A' . str_repeat(' ', 0),
    str_repeat(' ', 1) . 'B' . str_repeat(' ', 1),
    str_repeat(' ', 0) . 'C' . str_repeat(' ', 2),
];
```
Les tests passent toujours...

# Refacto #7 : Des lettres aux lignes

Les similitudes entre les lignes du quart supérieur gauche du diamant peuvent être utilisées pour le générer à partir d'une séquence de l'alphabet et d'une boucle pour ajouter les espaces en préfixe et suffixe . Pour le cas du diamant 'C' :
```php
$alphabet = range('A', 'C');
$size = count($alphabet);
$upperLeft = array_map(
    function ($letter, $i) use ($size) {
        return str_repeat(' ', $size - ($i +1)) . 
            $letter .
            str_repeat(' ', $i);
    },
    $alphabet,
    array_keys($alphabet);
);
```
Finalement, le nom `$letter`, que j'ai initialement choisi pour l'argument de la fonction `create`, m'embête maintenant. Je veux l'utiliser ici, et n'ai pas vraiment d'autre idée de nom pour ce contexte.  
Alors, même si les deux variables sont dans des contextes différents et n'interfèrent pas, j'en profite pour renommer l'argument en `$lastLetter`.  
Cela nous donne le code générique suivant :
```php
public function create($lastLetter) {
    $alphabet = range('A', $lastLetter);
    $size = count($alphabet);
    $upperLeft = array_map(
        function ($letter, $i) use ($size) {
            return str_repeat(' ', $size - ($i +1)) . 
                $letter .
                str_repeat(' ', $i);
        },
        $alphabet,
        array_keys($alpahbet)
    );
    $upper = array_map(
        function ($left) {
            return $left . substr(strrev($left), 1);
        },
        $upperLeft
    );
    $diamond = array_merge(
        $upper,
        array_slice(array_reverse($upper), 1)
    );
    return implode("\n", $diamond);
}
```

# Refacto 8 : Appels en cascade.

Pour construire le diamant, il faut d'abord construire le haut du diamant.  
Pour construire le haut du diamant, il faut d'abord construire le quart haut gauche du diamant.  
Cela ne me semble pas très logique de laisser le code séquentiellement dans la méthode `create`, je préfère l'organiser en sous-méthode, `createUpper` et `createUpperLeft`, appelées en cascade :  
`create` -> `createUpper` -> `createUpperLeft`.

```php
class Diamond {
    public function create($lastLetter) {
        $upper = $this->createUpper($lastLetter);
        $diamond = array_merge(
            $upper,
            array_slice(array_reverse($upper), 1)
        );
        return implode("\n", $diamond);
    }

    private function createUpper($lastLetter) {
        return array_map(
            function ($left) {
                return $left . substr(strrev($left), 1);
            },
            $this->createUpperLeft($lastLetter);
        );
    }

    private function createUpperLeft($lastLetter) {
        $alphabet = range('A', $lastLetter);
        $size = count($alphabet);
        return array_map(
            function ($letter, $i) use ($size) {
                return str_repeat(' ', $size - ($i +1)) . 
                    $letter .
                    str_repeat(' ', $i);
            },
            $alphabet,
            array_keys($alphabet)
        );
    }
}
```
