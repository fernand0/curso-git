## *Hooks*: ejecutando código tras una orden git
### Objetivos
### Viendo las cañerías: estructura de un repositorio `git`

Cuando se crea por primera vez un repositorio veremos que aparecen misteriosamente una serie de ficheros con esta estructura dentro del directorio `.git`.

![Estructura básica de un repositorio git](img/tree-git.png)

`branches` lo dejamos de lado porque ya no se usa (aunque por alguna razón se sigue creando). `config`, `HEAD`, `refs` y `objects` son ficheros o directorios que almacenan información dinámica, por ejemplo `config` almacena las variables de configuración (que se han visto anteriormente). El resto de los ficheros y directorios se copian de una *plantilla*; esta plantilla se instala con `git` y se usa cada vez que hacemos `clone` o `init`, y contiene `hooks`, `description`, `branches` e `info` y los ficheros que se encuentran dentro de ellos. 

Esta plantilla la podemos modificar y cambiar. La plantilla que se usa por omisión se encuentra en `/usr/share/git-core/templates/` y contiene una serie de ficheros junto con ejemplos (*samples*) para ganchos. Sin embargo, podemos personalizar nuestra plantilla editando (con permiso de superusuario) estos ficheros o bien usando la opción `--template <nombre de directorio>` de `clone` o `init`. En ese caso, en vez de copiar los ficheros por omisión, copiará los contenidos en ese directorio.

Por ejemplo, se puede usar [esta plantilla](http://jj.github.io/repo-plantilla) que elimina los ficheros de ejemplo, sustituye por otro y traduce los contenidos de los otros ficheros al castellano; también mete en los patrones ignorados (sin necesidad de usar `.gitignore`) los ficheros que terminan en `~`, que produce Emacs como copia de seguridad.

Estos ficheros forman parte de las cañerías de `git` y podemos cambiar su comportamiento editando `config` como ya se ha visto en el capítulo de uso básico; de hecho, existe también un fichero de configuración a nivel global, `.gitconfig` que sigue el mismo formato y que ya hemos visto

```
[alias]
	ci = commit
	st = status
[core]
	editor = emacs
```

Estos ficheros de configuración siguen un formato similar al de los ficheros `.ini`, es decir, bloques definidos entre corchetes y variables con valor, dentro de ese bloque, a las que se le asigna usando `=`. En este caso [definimos dos alias](http://wildlyinaccurate.com/useful-git-configuration-items) y un editor o, mejor dicho, *el* editor. Esto podemos hacerlo tanto en el fichero global como en el local si queremos que afecte sólo a nuestro repositorio.

Otro fichero dentro de este directorio que se puede modificar es `.git/info/exclude`; es similar a `.gitignore`, salvo que en este caso afectará solamente a nuestra copia local del repositorio y no a todas las copias del mismo. Por ejemplo, podemos editerlo de esta forma

```
# git ls-files --others --exclude-from=.git/info/exclude
# Lines that start with '#' are comments.
# For a project mostly in C, the following would be a good set of
# exclude patterns (uncomment them if you want to use them):
# *.[oa]
*~
``` 
para excluir los ficheros de copia de seguriad de Emacs (que hemos definido antes como editor) que nos interesa evitar a nosotros, pero que puede que tengan un significado especial para otro usuario del repo y que por tanto no quiera evitar. 

Por supuesto, el tema principal de este capítulo está en el otro directorio, *hooks*, cuyo contenido tendremos que cambiar si queremos añadir ganchos al repositorio. Pero para usarlo necesitamos también conocer algunos conceptos más de git, empezando por cómo se accede a más cañerías. 

### Paso a paso

Si miramos en el directorio `.git/objects` encontraremos una serie de
directorios con nombres de dos letras, dentro de los cuales están los
objetos de git, que tienen otras 38 letras para componer las 40 letras
que componen el nombre único de cada objeto; este nombre se genera a
partir del contenido usando
[SHA1](http://en.wikipedia.org/wiki/SHA-1). 

`git` almacena toda la información en ese directorio y de hecho está
[organizado para acceder a la información almacenada por contenido](http://git-scm.com/book/en/Git-Internals). Por
eso tenemos que imaginarlo como un sistema de ficheros normal, con una
raíz (que es HEAD, el punto en el que se encuentra el repositorio en
este momento) y una serie de ramas que apuntan a ficheros y a
diferentes versiones de los mismos.

git, entonces, [procede de la forma siguiente](http://git-scm.com/book/en/Git-Internals-Git-Objects).

1. Crea un SHA1 a partir del contenido del fichero cambiado o añadido. Este fichero
se almacena en la zona temporal en forma de *blob*. 
2.  El nombre del fichero se almacena en un árbol, junto con el nombre
de otros ficheros que estén, en ese momento, en la zona temporal. Un
árbol almacena los nombres de varios ficheros y apunta al contenido de
los mismos, almacenado como *blob*. De este objeto, que tiene un
formato fijo, se calcula también el SHA1 y se almacena en
`.git/objects`. Un árbol, a su vez, puede apuntar a otros árboles
creados de la misma forma.
3. Cuando se hace *commit*, se crea un tercer tipo de objeto con ese
nombre. Un *commit* contiene enlaces a un árbol (el de más alto nivel)
y metadatos adicionales: quién lo ha hecho, cuando y por supuesto el
mensaje de commit. 

Veremos más adelante cómo se listan ficheros de todos estos tipos,
pero por lo pronto la idea es que un comando de git de alto nivel
involucra varias órdenes de bajo nivel que, eventualmente, van a parar
a información que se almacena en un directorio determinado con nombres
de fichero que se calculan usando SHA1, aparte de que se puede actuar
a diferentes niveles, desde el más bajo de almacenar un objeto
directamente en un árbol o crear un commit "a mano" hasta el más alto
(que es el que estamos acostumbrados). Este sistema, además, asegura
que no se pierda ninguna información y que podamos acceder al
contenido de un fichero determinado hecho en un momento determinado de
forma fácil y eficiente. Pero para poder hacerlo debe haber una forma
única y también compacta de referirse a un elemento determinado dentro
de ese repositorio. Es lo que explicaremos a continuación.

### El nombre de las cosas: refiriéndonos a objetos en git.

Como ya hemos visto antes, todos los objetos (sean *blobs*, árboles o
*commits*) están representados por un SHA1.  Si conocemos el SHA1, se
puede usar `show`, por ejemplo, para visualizarlo. Haciendo `git log`
veremos, por ejemplo, los últimos commits y si hacemos `show` sobre
uno de ellos,

```
git show fe88e5eefff7f3b7ea95be510c6dcb87054bbcb
commit fe88e5eefff7f3b7ea95be510c6dcb87054bbcb0
Author: JJ Merelo <jjmerelo@gmail.com>
Date:   Thu Apr 17 18:29:11 2014 +0200
    Añade layout
diff --git a/views/layout.jade b/views/layout.jade
new file mode 100644
index 0000000..36cc059
--- /dev/null
+++ b/views/layout.jade
@@ -0,0 +1,6 @@
[....]
```

El mismo resultado que obtendríamos si hacemos `git show HEAD`, que
recordemos que es una referencia que apunta al último commit.  También
obtendremos lo mismo si hacemos `git show master`.  En cualquiera de
los casos, lo que está mostrando es un objeto de tipo *commit*, el
último realizado. 


Pero veremos como
funciona este último ejemplo. Al lado del directorio `objects` está el
directorio `refs`, que almacena referencias y que es como `git` sabe a
qué commit corresponde cada cosa. Este comando:

```~/txt/docencia/repo-tutoriales/repo-ejemplo<master>$ tree .git/refs/
.git/refs/
├── heads
│   ├── img-dir
│   └── master
├── remotes
│   ├── heroku
│   │   └── master
│   └── origin
│       ├── HEAD
│       ├── img-dir
│       └── master
└── tags
    ├── v0.0.1
    ├── v0.0.2
    ├── v0.0.2.1
    └── v0.0.3
5 directories, 10 files
```

muestra todo lo que hay almacenado en este directorio: referencia a
las ramas locales en `heads` y a las remotas en `remotes`. Si
mostramos el contenido de los ficheros:

```
~/txt/docencia/repo-tutoriales/repo-ejemplo<master>$ cat .git/refs/heads/master 
fe88e5eefff7f3b7ea95be510c6dcb87054bbcb0
```

Que muestra que, efectivamente, el hash del commit es el que
corresponde 

>Podemos mirar en .git/objects/fe a ver si efectivamente se encuentra;
> puedes hacerlo sobre tu copia del repositorio `repo-ejemplo`, ya que
> los hash son iguales en todos lados.

Como hemos visto anteriormente, un *commit* apunta a un árbol. Podemos
indicarle a `show` que nos muestre este árbol de esta forma:

```
~/txt/docencia/repo-tutoriales/repo-ejemplo<master>$ git show master^{tree}
tree master^{tree}
.aspell.es.pws
.gitignore
.gitmodules
.travis.yml
LICENSE
Procfile
README.md
curso
package.json
shippable.yml
test/
views/
web.js
```

En este caso el [formato es rama (circunflejo o *caret*) `{tree}`](https://gist.github.com/wfarr/1609626); el
circunflejo se usa en la selección de referencias de `git` para
cualificar lo que se encuentra antes de ella, pero no hay muchas más
opciones aparte de `tree`, pero sí podemos acceder a versiones
anteriores del repositorio y a sus ficheros. 

```
~/txt/docencia/repo-tutoriales/repo-ejemplo<master>$ git show master~1
commit 5be23bb2a610260da013fcea807be872a4bd6981
Author: JJ Merelo <jjmerelo@gmail.com>
Date:   Thu Apr 17 17:42:39 2014 +0200
    Aclara una frase
[...]
```

La [tilde `~`](http://www.vogella.com/tutorials/Git/article.html#commitreference) indica un ancestro, es decir, el *padre* del commit
anterior, que, como vemos
[corresponde al commit 5be23bb](https://github.com/oslugr/repo-ejemplo/commit/5be23bb2a610260da013fcea807be872a4bd6981). Podemos
ir más allá hasta que nos aburramos: `~2` accederá al padre de este y
así sucesivamente. Y, por supuesto, podemos cualificarlo con
`^{tree}^` para que nos muestre el árbol en el estado que estaba en
ese commit. Y también para que nos muestre un fichero sin necesidad de
sacarlo del repositorio:

```
~/txt/docencia/repo-tutoriales/repo-ejemplo<master>$ git show master~2:README.md 
repo-ejemplo
============
Ejemplo de repositorio para trabajar en el
[curso de `git`](http://cevug.ugr.es/git) el contenido del cual está
[...]
```

>En esta sección hemos usado `show` para mostrar las capacidades de
> los diferentes selectores de `git`, pero se pueden usar con
> cualquier otra orden, como `checkout` o cualquiera que admita, en
> general, una referencia a un objeto.



### Comandos de alto y bajo nivel: *fontanería* y *loza*
En este caso tenemos objetos de tres tipos: blob, commit y tree. a `ls-tree` se le pasa un *tree-ish*, es decir, algo que apunte a dónde esté almacenado un árbol pero, para no preocuparnos de qué se trata esto, usaremos simplemente HEAD, que apunta como sabéis a la punta de la rama en la que nos encontramos ahora mismo. También  nos da el SHA1 de 40 caracteres que representa cada uno de los ficheros. Si queremos que se expandan los `tree` para mostrar los ficheros que hay dentro también usamos la opcion `-r`

```
~/txt/docencia/repo-tutoriales/repo-ejemplo<master>$ git ls-tree -r HEAD
100644 blob a6f69e4284566cd84272c6a4e4996f64643afbea	.aspell.es.pws
100644 blob a72b52ebe897796e4a289cf95ff6270e04637aad	.gitignore
100644 blob cc5411b5557f43c7ba2f37ad31f8dc34cccda075	.gitmodules
100644 blob 4e7b6c1b5a6cb3a962ea05874d10c943c1923f39	.travis.yml
100644 blob d5445e7ac8422305d107420de4ab8e1ee6227cca	LICENSE
100644 blob d1913ebe4d9e457be617ee0e786fc8c30a237902	Procfile
100644 blob da5b5121adb42e990b9e990c3edb962ef99cb76a	README.md
160000 commit fa8b7521968bddf235285347775b21dd121b5c11	curso
100644 blob f8c35adaf57066d4329737c8f6ec7ce6179cc221	package.json
100644 blob 08827778af94ea4c0ddbc28194ded3081e7b0f87	shippable.yml
100644 blob 9920d80438d42e3b0a6924a0fcace2d53a6af602	test/route.js
100644 blob 36cc059186e7cb247eaf7bfd6a318be6cffb9ea3	views/layout.jade
100644 blob 97c32024cda29e0fb6abebf48d3f6740f0acb9e2	web.js
``` 
que muestra solo los objetos de tipo `blob` (y un `commit`) con el camino completo que llega hasta ellos. 

Si editamos un fichero tal como el README.md, tras hacer el commit tendrá esta apariencia:
Hay un tercer comando relacionado con el examen de directorios y ficheros locales, [`cat-file`, que muestra el contenido de un objeto](http://git-scm.com/docs/git-cat-file), en general. Por ejemplo, en este caso, para listar el contenido de un objeto de tipo `tree`
nos muestra que ese objeto contiene un solo fichero, `layout.jade`, y sus características. Pero más curioso aún es cuando se usa sobre objetos de tipo *commit* (no sobre el objeto *commit* que aparece arriba, que se trata de un *commit* de *otro repositorio* al contener el directorio `curso` un submódulo de git. Por ejemplo, podemos hacer:

```
~/txt/docencia/repo-tutoriales/repo-ejemplo<master>$ git cat-file -p HEAD
tree 1c40899a32c2b5ec7f930bd943e5dbb98562d373
parent 5be23bb2a610260da013fcea807be872a4bd6981
author JJ Merelo <jjmerelo@gmail.com> 1397752151 +0200
committer JJ Merelo <jjmerelo@gmail.com> 1397752151 +0200
Añade layout
```

que, dado que `HEAD` apunta al último commit, nos puestra *pretty-print* toda la información sobre el último *commit* y muestra el árbol de ficheros correspondiente, que podemos listar con 

```
~/txt/docencia/repo-tutoriales/repo-ejemplo<master>$ git cat-file -p 1c40899a
100644 blob a6f69e4284566cd84272c6a4e4996f64643afbea	.aspell.es.pws
100644 blob a72b52ebe897796e4a289cf95ff6270e04637aad	.gitignore
100644 blob cc5411b5557f43c7ba2f37ad31f8dc34cccda075	.gitmodules
100644 blob 4e7b6c1b5a6cb3a962ea05874d10c943c1923f39	.travis.yml
100644 blob d5445e7ac8422305d107420de4ab8e1ee6227cca	LICENSE
100644 blob d1913ebe4d9e457be617ee0e786fc8c30a237902	Procfile
100644 blob da5b5121adb42e990b9e990c3edb962ef99cb76a	README.md
160000 commit fa8b7521968bddf235285347775b21dd121b5c11	curso
100644 blob f8c35adaf57066d4329737c8f6ec7ce6179cc221	package.json
100644 blob 08827778af94ea4c0ddbc28194ded3081e7b0f87	shippable.yml
040000 tree 39da6b155c821af1e6a304daca9b66efb1ac651f	test
040000 tree fd3846c0d6089437598004131184c61aea2b6514	views
100644 blob 97c32024cda29e0fb6abebf48d3f6740f0acb9e2	web.js
```

En general, si queremos ahondar en las entrañas de un punto determinado en la historia del repositorio, trabajar con `ls-files`, `cat-file` y `ls-tree` permite obtener toda la información contenida en el mismo. Esto nos va a resultar útil un poco más adelante. 


### Concepto de *hooks*
### Programando un *hook* básico
### Algunos *hooks* útiles explicados