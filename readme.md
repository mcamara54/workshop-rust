## INSTALLATION

### Installer RUST sur Linux:
```
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```
Exécuter l'**installation par défaut** en séléctionnant 1 puis **relancez** le terminal pour mettre à jour le PATH.

### Ensuite, run:
```
source "$HOME/.cargo/env"
```

Pour vérifier que tout est bien **installé**, exécutez la commande:
```
cargo --version
```

## CRÉER SON PROJET RUST

### Création:
```
cargo new nom_projet
```
Ceci va initier un nouveau projet avec le nom choisit. Cela va créer un fichier **"Cargo.toml"** dans lequel se trouveront toutes nos **dépendances** et un fichier **"src/main.rs"** dans lequel on va travailler.

### Test:
```
cd nom_projet
cargo run
```
Le projet devrait maintenant compiler et renvoyer "Hello, world!" en **sortie**.

### Output:
```
   Compiling workshop-rust v0.1.0 (/path/to/rust/project/nom_projet)
    Finished dev [unoptimized + debuginfo] target(s) in 0.20s
     Running `target/debug/workshop-rust`
Hello, world!
```
Le projet fonctionne bien, on peut maintenant passer à l'objectif du projet: faire un **jeu** de type en **hunter**.

### Librairie graphique:

On utilisera la sdl2 sur RUST.
```
sudo apt-get install -y libsdl2-dev
sudo apt-get install -y libsdl2-image-dev
sudo apt-get install -y libsdl2-ttf-dev
```

## IMPLÉMENTER SON HUNTER

 Ajouter la dépendance SDL2 dans le "**Cargo.toml**":

```
[dependencies]
sdl2 = "*"
```

Dans le fichier "**src/main.rs**", il faut ajouter les différents **imports** nécessaires pour la SDL2:

```
extern  crate sdl2;
use sdl2::event::Event;
use sdl2::keyboard::Keycode;
use core::time::Duration
```

On peut maintenant **initialiser sa fenêtre**:

```
const WINDOW_WIDTH: u32 = 800;
const WINDOW_HEIGHT: u32 = 600;

fn  main() {
	let  sdl_context  = sdl2::init().unwrap();
	let  video_subsystem  =  sdl_context.video().unwrap();
	let  window  =  video_subsystem.window("Aim Hunter Rust", WINDOW_WIDTH, WINDOW_HEIGHT)
		.position_centered()
		.build()
		.unwrap();

	let  mut  canvas  =  window.into_canvas().build().unwrap();
	let  mut  event_pump  =  sdl_context.event_pump().unwrap();

	'running:  loop {
		for  event  in  event_pump.poll_iter() {
			match  event {
				Event::Quit { .. } |  Event::KeyDown { keycode:  Some(Keycode::Escape), .. } =>  break 'running,
				_  => {}
			}
		}
		::std::thread::sleep(Duration::new(0, 1_000_000_000u32  /  60));
	}
}
```

De cette manière, on obtient une fenêtre toute noire en 800 par 600 et **fermable** en cliquant sur **ESC**.
On va maintenant ajouter les éléments nécessaires pour la création de notre jeu.

### Structures:

 - Piafs
```
struct Bird {
	position: Point,
	alive: bool,
}
```
 - Game
```
struct Game {
	canvas: Canvas<Window>,
	birds: Vec<Bird>,
	score: u32,
	clicks: u32,
	start_time: Instant,
	clicks_in: u32,
}
```

### Implémentation des fonctions nécessaire à la structure Game:

```
impl Game {
	// ici vont être les fonctions
}
```

La première sera le **constructeur**.
```
fn  new(canvas:  Canvas<Window>) ->  Game {
	Game {
		canvas,
		birds:  Vec::new(),
		score:  0,
		clicks:  0,
		start_time:  Instant::now(),
		clicks_in:  0,
	}
}
```

On aura également besoin d'une fonction pour faire **spawner** les "oiseaux".

```
fn  spawn_bird(&mut  self) {
	let  x  = rand::thread_rng().gen_range(0..WINDOW_WIDTH  as  i32  -  BIRD_SIZE);
	let  y  = rand::thread_rng().gen_range(0..WINDOW_HEIGHT  as  i32  -  BIRD_SIZE);
	let  bird  =  Bird {
	position:  Point::new(x, y),
	alive:  true,
	};
	self.birds.push(bird);
}
```

Une fonction **update** pour faire **bouger** les piafs et faire **respawn** ceux décédey.

```
fn  update(&mut  self) {
	let  mut  rng  = rand::thread_rng();
	for  bird  in  &mut  self.birds {
		if  bird.alive {
			let  x_offset  =  rng.gen_range(-2..=2);
			let  y_offset  =  rng.gen_range(-2..=2);
			bird.position.x +=  x_offset;
			bird.position.y +=  y_offset;
		}
	}
	self.birds.retain(|bird|  bird.alive);
	while  self.birds.len() < MAX_BIRDS {
		self.spawn_bird();
	}
}
```

Une fonction **draw** qui va **afficher** tous les éléments dans la fenêtre.

```
fn  draw(&mut  self) {
	self.canvas.set_draw_color(Color::RGB(0, 0, 0));
	self.canvas.clear();
	self.canvas.set_draw_color(Color::RGB(255, 0, 0));
	for  bird  in  &self.birds {
		if  bird.alive {
			self.canvas.fill_rect(Rect::new(bird.position.x, bird.position.y, BIRD_SIZE  as  u32, BIRD_SIZE  as  u32)).unwrap();
		}
	} 
	self.canvas.present();
}
```

La fonction **handle_click** qui va donc vérifier si les piafs vont **crever ou non**.

```
fn  handle_click(&mut  self, x:  i32, y:  i32) {
	self.clicks +=  1;
	for  bird  in  &mut  self.birds {
		if  bird.alive {
			if  x  >=  bird.position.x &&  x  < bird.position.x +  BIRD_SIZE  && y  >=  bird.position.y &&  y  <  bird.position.y +  BIRD_SIZE {
				bird.alive =  false;
				self.score +=  10;
				self.clicks_in +=  1;
				break;
			}
		}
	}
}
```

La fonction **game_duration_elapsed** qui vérifie si le **temps de jeu** est **écoulé**.

```
fn  game_duration_elapsed(&self) ->  bool {
	self.start_time.elapsed().as_secs() >= GAME_DURATION
}
```

Pour finir avec Game, il faut la fonction **calculate_precision** qui va permettre d'**afficher au joueur** sa **précision** sur la partie.

```
fn  calculate_precision(&self) ->  f32 {
	return  self.clicks_in as  f32  *  100.0  / self.clicks as  f32;
}
```

### Mise à jour du "Cargo.toml":

Il faut ajouter la dépendance de **rand**. Le fichier devrait donc maintenant ressembler à cela.
```
[dependencies]
sdl2 = "*"
rand = "0.8.5"
```
### Mise à jour des imports:
```
use sdl2::event::Event;
use sdl2::keyboard::Keycode;
use sdl2::mouse::MouseButton;
use sdl2::pixels::Color;
use sdl2::rect::{Rect, Point};
use sdl2::render::Canvas;
use sdl2::video::Window;
use rand::Rng;
use std::time::{Duration, Instant};

const  WINDOW_WIDTH:  u32  =  800;
const  WINDOW_HEIGHT:  u32  =  600;
const  BIRD_SIZE:  i32  =  20;
const  MAX_BIRDS:  usize  =  10;
const  GAME_DURATION:  u64  =  30; // Durée du jeu en secondes
```

### Modification en conséquence du main:

On ajoute l'**instance** de Game:
```
let mut game = Game::new(canvas);
```

On ajoute l'évènement **handle_client** dans la **gestion des évènements** de la boucle principale:

```
Event::MouseButtonDown { x, y, mouse_btn, .. } => {
	if  mouse_btn  ==  MouseButton::Left {
		game.handle_click(x, y);
	}
}
```

On ajoute les **conditions** vérifiant si le **temps et écoulé** ou non et qui agissent en conséquences !

```
if  !game.game_duration_elapsed() {
	if  game.birds.len() < MAX_BIRDS {
		game.spawn_bird();
	}
	game.update();
	game.draw();
} else {
	let  accuracy  = game.calculate_precision();
	println!("Score: {}, accuracy: {:.2}", game.score, accuracy);
	break 'running;
}
```

## Axes d'améliorations

Maintenant que le jeu est opériationnel, bien qu'à chier, c'est à vous de l'améliorer en y ajoutant vos touches personnelles telles que des sprites pour les oiseaux, un background, un affichage du score, de l'accuracy, et plein d'autres choses.

Bisous, coordialement nous!