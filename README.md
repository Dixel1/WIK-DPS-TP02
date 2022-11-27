# WIK-DPS-TP02

Ce TP a été effectué en se basant sur le code du repositorie https://github.com/Sarvagon/WIK-DPS-TP01-Rust-Version

# Optimisation des images docker.

## 1 - Docker Single stage for the rust api :

### 1.1 - Methode 1 : Utilisation du buildkit de docker


> Pour optimiser le temps de build, on va mettre en cache les dossiers de build :

```dockerfile
FROM rust
ENV PING_LISTEN_PORT=8080
ENV HOME=/home/root
WORKDIR  $HOME/app
ADD src src
ADD Cargo.lock .
ADD Cargo.toml .
RUN --mount=type=cache,target=/usr/local/cargo/registry \
    --mount=type=cache,target=/home/root/app/target \
    cargo build --release 
EXPOSE $PING_LISTEN_PORT
CMD ["cargo", "run", "--release"]
```



> Une fois l'image ecrite on demande à docker de construire l'image :

```bash
DOCKER_BUILDKIT=1 docker build --progress=plain --tag rust-api .
```

La commande de lancement de l'image est :

```bash
docker run -p 8080:8080 rust-api
```

Taille de l'image : 1.34GB




### Test de l'image

Pour tester si l'image est plus performante, on va comparer avec une autre image qui ne met pas en cache les dossiers de build.

```dockerfile
FROM rust
ENV PING_LISTEN_PORT=8080
ENV HOME=/home/root
WORKDIR  $HOME/app
ADD src src
ADD Cargo.lock .
ADD Cargo.toml .
RUN cargo build --release 
EXPOSE $PING_LISTEN_PORT
CMD ["cargo", "run", "--release"]
```

On build une première fois les images, puis on modifie le code source puis on re-build l'image pour mesurer le temps.



On mesure le temps de build de l'image sans cache :
 ```bash 
 time docker build --progress=plain --tag rust-api .
 ```
on optient le temps suivant : `real    0m 2.64s`

Maintenant on mesure le temps de build de l'image avec cache :

```bash
time sudo DOCKER_BUILDKIT=1 docker build --progress=plain --tag rust-api .
```
Cette fois ci on obtient le temps suivant : `real    0m 0.51s`

Soit une différence de temps de build de 2.13s pour un projet simple.
Pour des projets plus gros, la différence de temps de build peut être encore plus importante. vu que les dépendances sont plus nombreuses.

 

### 1.2 - Methode 2 : compilation des dépendances avec un proramme vide

> Pour optimiser le temps de build, cette fois ci on va compiler les dépendances dans un programme vide.puis dans un autre layer on va compiler le programme principal.

```dockerfile
FROM rust
ENV PING_LISTEN_PORT=8080
ENV HOME=/home/root
WORKDIR  $HOME/app
ADD Cargo.lock .
ADD Cargo.toml .
ADD dummy.rs .
RUN sed -i 's#src/main.rs#dummy.rs#' Cargo.toml 
RUN cargo build --release 
RUN sed -i 's#dummy.rs#src/main.rs#' Cargo.toml
ADD src src
RUN cargo build --release 
EXPOSE $PING_LISTEN_PORT
CMD ["cargo", "run", "--release"]
```

avec dummy.rs :

```rust
fn main() {}
```

puis on rajoute dans cargo.toml :

```toml
[[bin]]
name = "app"
path = "src/main.rs"
```


### Test de l'image

On utilise la meme image basique de test que pour la methode 1. qui avait eu un temps de build de 2.64s

avec la methode 2, on obtient un temps de build de 1,5s soit une différence de 1.14s.

Cette fois ci la méthode 2 est moins performante que la méthode 1. Mais elle évite d'utiliser le buildkit de docker qui est en version experimental.
et encore une fois, pour des projets plus gros, la différence de temps de build peut être encore plus importante. 

 **Explication :** 
 dans notre dockerfile, on realise des sed pour modifier le fichier cargo.toml afin de premierement compiler "dummy.rs" puis "main.rs".
 
## 2 - Docker multi stage for the rust api 

### 2.1 - Methode 1 : 

Dans cette methode, on va utiliser un dockerfile multi stage. On va donc compiler le programme dans un premier stage puis on va copier le binaire dans un stage final.
Cela permet d'avoir un dockerfile plus simple et plus lisible. en plus on peut utiliser des images plus légères pour le stage final.
On pourrait utiliser les meme methode d'optimisation que précédemment, mais ici on va utiliser cargo-chef.
Cargo-chef est un compilateur visant a optimiser les build, mais on ne la pas utilisée avant car sont utilisations ce prête moins bien au dockerfile single-stage.


```dockerfile
FROM rust as planner
WORKDIR app
RUN cargo install cargo-chef
COPY . .
RUN cargo chef prepare --recipe-path recipe.json

FROM rust as cacher
WORKDIR app
RUN cargo install cargo-chef
COPY --from=planner /app/recipe.json recipe.json
RUN cargo chef cook --release --recipe-path recipe.json

FROM rust as builder
WORKDIR app
COPY . .
COPY --from=cacher /app/target target
RUN cargo build --release 

FROM debian:buster-slim
WORKDIR app
ENV PING_LISTEN_PORT=8080
COPY --from=builder /app/target/release/cour_dev_ops .
EXPOSE 8080
RUN ["./cour_dev_ops"]

```

**Explication :**
 - 1er stage : on installe cargo-chef puis on prepare le recipe.json qui contient les dépendances du projet.
 - 2eme stage : on installe cargo-chef puis on compile les dépendances avec le recipe.json
 - 3eme stage : on compile le programme principal
 - 4eme stage : on copie le binaire dans une image debian:buster-slim et on lance le programme.

pour build :

```bash
time docker build --progress=plain --tag rust-api .
```

pour run :

```bash
docker run -p 8080:8080 rust-api
```

### Test de l'image

Si on regarde la taille de nos images précédentes, elle font toutes plus de 1Go. cela est du au fait que l'image de base est rust:latest qui fait plus de 1Go.
Mais grace au multi-stage, et l'execution du programme dans une image debian:buster-slim, on obtient une image de 70Mo.
Pour le temps de build en cas de modification du code, on obtient un temps de build de 4,2s. Ce qui est bien moins rapide que la methode 1. mais on a une image plus légère.


*Lien du github de cargo-chef :* https://github.com/LukeMathWalker/cargo-chef

### 2.2 - Methode 2 : dummy.rs + Multi-stage

Comme vu précédemment, le temps de build etait plus long que la methode 1, mais cela est peu etre du au fait que les image multi-stage sont plus longue a build.
Pour verifier cela, on va créer une image comme celle de la methode 1.2 mais en utilisant un dockerfile multi-stage.

```dockerfile
FROM rust as builder
ENV HOME=/home/root
WORKDIR  $HOME/app
ADD Cargo.lock .
ADD Cargo.toml .
ADD dummy.rs .
RUN sed -i 's#src/main.rs#dummy.rs#' Cargo.toml 
RUN cargo build --release 
RUN sed -i 's#dummy.rs#src/main.rs#' Cargo.toml
ADD src src
RUN cargo build --release 

FROM debian:buster-slim
WORKDIR app
ENV PING_LISTEN_PORT=8080
COPY --from=builder /home/root/app/target/release/app .
EXPOSE 8080
ENTRYPOINT ["./app"]
```


on build :

```bash
time docker build --progress=plain --tag rust-api .
```

puis on run :

```bash
docker run -p 8080:8080 rust-api
```

Taille de l'image : 70Mo

Temps de re-build en cas de modification du code : 2,8 s

En effet, les images multi-stage sont plus longue a build que les images single-stage. Mais cela reste plus rapide que la methode 2.1

### 2.3 - Methode 3 :  Multi-stage + buildkit

Pour finir les test sur les images multi-stage, on va tester la methode 1.1 mais en multistage.

```dockerfile
FROM rust as builder
ENV HOME=/home/root
WORKDIR  $HOME/app
ADD src src
ADD Cargo.lock .
ADD Cargo.toml .
RUN --mount=type=cache,target=/usr/local/cargo/registry \
    --mount=type=cache,target=/home/root/app/target \
    cargo build --release \
    && cp target/release/app ./app

FROM debian:buster-slim
WORKDIR app
ENV PING_LISTEN_PORT=8080
COPY --from=builder /home/root/app/app .
EXPOSE $PING_LISTEN_PORT
ENTRYPOINT ["./app"]
```

on build :

```bash
time DOCKER_BUILDKIT=1 docker build --progress=plain --tag rust-api . 
```

Taille de l'image : 70Mo
Temps de re-build en cas de modification du code : 1s

notre nouvelle image est lègère et rapide a build. Vive le buildkit !

Tableau recapitulatif des images :

Image | Taille | Temps de build
---|---|---
Basique | 1,34Go | 2,64s
Buildkit | 1,34Go | 0,5s
Dummy.rs | 1,34Go | 1,5s
Multi-stage  + cargo-chef | 70Mo | 4,2s
Multi-stage + dummy | 70Mo | 2,8s
Multi-stage + buildkit | 70Mo | 1s



### Conclusion

Globalement, les images multi-stage sont plus longue a build que les images single-stage. Mais 
l'utilisation du buildkit permet de gagner du temps sur le build et cela meme avec une image multi-stage.

On utilisera donc la derniere methode pour notre image docker.

## 2 - Suite du TP

### Utilisateur spécifique

Pour utiliser un utilisateur spécifique dans notre image, on va créer un utilisateur dans notre image debian puis l'utiliser 
avec l'instruction USER.


```dockerfile
FROM rust as builder
ENV HOME=/home/root
WORKDIR  $HOME/app
ADD src src
ADD Cargo.lock .
ADD Cargo.toml .
RUN --mount=type=cache,target=/usr/local/cargo/registry \
    --mount=type=cache,target=/home/root/app/target \
    cargo build --release \
    && cp target/release/app ./app

FROM debian:buster-slim
RUN useradd -m -d /home/api api && chown -R api:api /home/api
USER api
WORKDIR app
ENV PING_LISTEN_PORT=8080
COPY --from=builder /home/root/app/app .
EXPOSE $PING_LISTEN_PORT
ENTRYPOINT ["./app"]
```

On peu vérifier que l'utilisateur est bien utilisé en regardant le processus lancé dans le container :

```bash
docker run -p 8080:8080 rust-api
```

```bash
> ps aux

USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
api          1  0.0  0.0   4336   728 ?        Ss   15:05   0:00 ./app
```

### Analyse de l'image 

Analyse de l'image en utilisant Trivy :
    
```bash 
sudo triy rust-api
```


Trivy rapporte énormament de failles de notre image : LOW: 10, MEDIUM: 47, HIGH: 42, CRITICAL: 5