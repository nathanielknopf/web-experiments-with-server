#Cocolab Experiment Game

This is a framework for online experiments meant for use on Amazon's MTurk platform for data collection. It is based off of [feste's experiment template repo](https://github.com/feste/experiment_template), with an added nodejs server that allows for additional control and manipulations over the experiment. 

The repo includes a small game made with Phaser JS in which the subject explores a world and interacts with objects. The game allows for easy implementation of causal systems, but can be designed to allow for different worlds and experiments.

##Running the Experiment

Clone this repo, `cd` into the directory, and run `npm install` to install the dependencies locally. 

Various parameters for the server and for the experiment can be configured with the configs object in the the config.js file.

By default, the server tries to run HTTPS on port 4343, and if it fails it falls back to HTTP on port 8080. It also defaults to not logging in a mysql database.
By default, the game runs for 1 minute with a 50% chance of either condition a or condition b, with small and large rocks spawning with equal likelihood.

To run the server:

```
node server.js
```

To access the experiment from the server machine, point your browser to `localhost:<port>/experiment.html`

##How it works
Information about how the experiment works

###The Server
The server for the experiment is built with express, socket.io, node-mysql, and a package called moment which is used for generating timestamps.

The server uses socket.io to receive information from the client about actions the subject has done. Socket.io can also be used to trasnmit parameters of the experiment to the client. 

The server pipes subjects through three pages - experiment.html, which describes the experiment and includes legal information, game.html, which contains the actual game the subject plays, and exitsurvey.html, which collects information from the subject after they've completed the game.

Additionally, there is a config.js file which includes an object with several parameters that can be defined to tweak how the experiment is run, such as the length of the game, as well as some properties of the web server and the database. This file is loaded by the server script, which expects to find it within its local directory.

###The Database
Results from the web experiment are stored in a sql database using mysql and a node package called node-mysql which interfaces node with mysql. The file at /js/databse.js contains a function for adding a player to the database (by default, players are identified by their MTurk worker ID, and have their results logged in a table named with their worker ID). 

The file also contains a function for inserting new data from the experiment into the players results table. These functions, which are ultimately called by the server.js script, can be changed as desired by the experimenter to log different parameters of the experiment.

###The Game
Phaser is a JS Framework for rapidly developing in-browser games.  Within the game.html file, there is a script in which the Phaser game runs. At the beginning of the script, a game instance is called with the following line of code:
```javascript
var game = new Phaser.Game(width, height, Phaser.AUTO, '', { preload: preload, create: create, update: update });
```

Following the declaration of variables, Phaser requires three main functions: preload, create, and update. Preload is where assets required by the game are loaded, create is where elements of the game are created and added, and update is a function that is continously called during gameplay where things like collision detection and key press listeners are called.

#####Preload
Within the preload function, all of the images used by Phaser are loaded and given names. These images are later used as sprites when instances of an object are created.

An example preload function that loads a background image, an image for a player character, and an image for a rock is shown below:
```javascript
function preload(){
	//lines of the form: game.load.image('name of image', 'path to image');
	game.load.image('background', 'background.png');
	game.load.image('player', 'player.img');
	game.load.image('rock', 'rock.img');
}
```

#####Create
Within the create function, Phaser creates a backbone for the game including a simple physics engine and listeners for keypresses. Additionally, within this function the experimenter can create all desired elements of the game world. In this repo, this includes all of the rocks, logs, trees, and ponds that form.

In general, individual objects on the screen are elements of a group of objects which has certain properties. New groups of objects can be added to the world by calling the `game.add.group()` function and storing the returned group in a variable. If there is a group stored in the `rocks` variable, a new instance of this group of objects can be created with the `rocks.create(x, y, 'rock')` function. The parameters of the function define its position on screen, as well as the image that it should use.

Objects can be created without being an instance of a group of objects as described above, but it is useful to use groups in scenarios where more than one instance of an object type will exist. This allows the experimenter to describe relationships between kinds of objects, rather than defining the relationships between each individual object on screen.

It is important to note that Phaser places the objects on the screen in the order that they are defined in the create function -- therefore, backgrounds should be created first, and elements that should be at the front of the screen, such as a player, should be created last.

An example create function could take the following form:
```javascript
function create(){
	game.physics.startSystem(Phaser.Physics.ARCADE) //Begin the simple physics engine within the game

	var background = game.add.sprite(0, 0, 'background'); //load a background image

	var rocks = game.add.group(); //create an object group called rocks
	rocks.enableBody = true; //physics engine listens for this group
	var rock = rocks.create(50, game.height-50, 'rock'); //place a rock in the bottom left corner of the screen
	rock.body.immovable = true; //fixes the rock's location -- prevents the player from moving it by bumping into it

	var player = game.add.sprite(0, 0, 'player') //add a player character to the screen in the upper right corner
	game.physics.arcade.enable(player) //physics engine listens for this object
	player.body.collideWorldBounds = true //keep the player within the boundaries of the game

	var cursors = game.input.keyboard.createCursorKeys() //tell Phaser that it will be listening for arrow key presses
```

#####Update
The update function contains code which Phaser will repeat as long as the game is running. This includes code for collision detection and listening for key presses. Allowing for the user of the game to control the character can be programmed within this funciton, as shown in the following example:
```javascript
function update(){
	game.physics.arcade.collide(player, rocks); //tell Phaser that the player should collide with instances of the rock group

	//unless we say differently later, the player character shouldn't move on its own
	player.body.velocity.x = 0;
    player.body.velocity.y = 0;

    //check the left and right arrow keys, move the player if they're pressed
    if(cursors.left.isDown || cursors.right.isDown){
    	if (cursors.left.isDown && cursors.right.isDown){ //if both left and right arrow keys are pressed, the player shouldn't move
    		player.body.velocity.x = 0;
    	} else if (cursors.right.isDown){
    		player.body.velocity.x = <speed>; //assign the player character a speed - positive speeds are to the right
    	} else if (cursors.left.isDown){
    		player.body.velocity.x = - <speed>; //a negative speed moves the player character to the left
    	}
    }

    //check the up and down arrow keys, move the player if they're pressed
    if(cursors.up.isDown || cursors.down.isDown){
    	if (cursors.up.isDown && cursors.down.isDown){
    		player.body.velocity.y = 0; //again, the player character shouldn't move if both up and down are pressed
    	} else if (cursors.down.isDown){
    		player.body.velocity.y = <speed>; //a positive speed moves the player character down the screen
    	} else if (cursors.up.isDown){
    		player.body.velocity.y = - <speed>; //a negative speed moves the player character up the screen
    	}
    }
}
```

If one wishes to implement causal relationships between objects or groups of objects, it is useful to associate a callback function with the collision between two groups of objects. In order to tell Phaser to call a callback function when it detects an object collision, one must include the following line of code within the update function:
```javascript
game.physics.arcade.overlap(group/object, group/object, callback, null, this);
```

The callback function can be defined somewhere outside of the preload, create, and update functions in normal javascript. An example implementation of a callback function being called when the player collides with a rock would look like the following. First, in the update function:
```javascript
game.physics.arcade.overlap(player, rocks, playerTouchRock, null, this);
```

Then, somewhere else, the callback function playerTouchRock, with two parameters -- the player object, and the rock object that the player collided with:
```javascript
function playerTouchRock(player, rock){
	var player_location = {x: player.x, y: player.y}
	socket.emit('rock collision', player_location)
}
```

In this example, the client communicates to the server using socket.io (as described in the web server section) by emitting a 'rock collision' event, with the player_location object as its data.

##Help

Please email nathanielknopf@gmail.com with any questions about this repo.
