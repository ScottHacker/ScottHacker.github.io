---
layout: post
title: "Light Block: a puzzle game in C#"
date: 2012-12-10
categories: project C#
---

LightBlock
==========

![LightBlock Board](http://iam.colum.edu/students/Scott.Hacker/blog/flatpress/fp-content/images/lightblock.png)

This is a puzzle game I'd wanted to try for a while, based on floor puzzles from the game Legend of Zelda.

The idea behind the puzzle is simple, the "player" is placed on a square on a floor.  The player then has to light up every square moving one space at a time from that position without crossing their own path.  Additionally, some squares are "blocks" where the player cannot move.  This sounds simple, but can be deceptively tricky.  Especially since I decided that I wanted to allow the player to choose the size of the board, and wanted to be able to construct the game-boards randomly.

[Mechanics Code](https://github.com/ScottHacker/LightBlock/blob/master/LightBlock/LightBlock.cs) | [GUI Code](https://github.com/ScottHacker/LightBlock/blob/master/LightBlock/GameForm.cs)

How I did it:
-------------

The first thing I had to figure out was how to construct the grid.  For starters, I decided to make a class for a "Square" type, which will hold a location and a state.

    public class Square
    {
        public Location location;
        public SquareState currentState;

        public Square(int x, int y, int size)
        {
            this.currentState = SquareState.Dark;
            this.location = new Location(x, y);
        }
    }

The state system is a simple enum, holding all the different states that a square could take.  This allows me to easily add states if I need to, as well as identify them by name.  I added a "Player" state, because I want the square the player is on to be a different color than the rest so that the player's position can be easily identified.

    public enum SquareState { Light, Dark, Block, Player }

I use a struct for Location, because I want this to be a custom type.  It's simple a 1-D array that holds two values, 0 being x and 1 being y.  I add some read only get accessors so that I can access these values easier by calling Location.x or Location.y.  Then I add a constructor which takes two ints as x and y, so that the usage of an array for this location is entirely invisible to the functions calling this type.

        public int x
        {
            get { return loc[0]; }
        }

        public int y
        {
            get { return loc[1]; }
        }

        public Location(int x, int y)
        {
            loc = new int[] { x, y };
        }

Then I add some overrides so that we can do some operations on this type.  I override the + operator so I can add locations, and I override ToString so that it'll print out the location in a clean and presentable way for the sake of debugging.

        public static Location operator +(Location l1, Location l2)
        {
            return new Location(l1.x + l2.x, l1.y + l2.y);
        }

        public override string ToString()
        {
            return "X: " + x + ", Y: " + y;
        }

I finish up by creating some pre-made Locations that represent movement directions.  This should come in use later.

        public static Location Left = new Location(-1, 0);
        public static Location Right = new Location(1, 0);
        public static Location Up = new Location(0, -1);
        public static Location Down = new Location(0, 1);
        public static List<Location> Directions = new List<Location>() { Left, Right, Up, Down };

Now that we have a Square class, a state system, and a Location type, we can actually move on to constructing the grid itself.  I make a new class called Grid which handles creating the grid and returning information about it.

I start with the class constructor, this gets the grid size from a static variable in another class called "LightBlockEngine" which I'll get to later.  It simply runs two For loops that create squares in the default "dark" state then adds them to the 2-D board array.

        public Grid()
        {
            board = new Square[LightBlockEngine.maxX, LightBlockEngine.maxY];

            for (int x = 0; x < LightBlockEngine.maxX; x++)
            {
                for (int y = 0; y < LightBlockEngine.maxY; y++)
                {
                    Square s = new Square(x, y, LightBlockEngine.blockSize);
                    board[x, y] = s;
                }
            }

Not done with the constructor yet, now that I've got a board with all the squares it needs with all of them set to the dark state, I need to change a few of them to the "block" state.  The number and placement of the block is random, so I set up a formula to figure out how many blocks a grid of the size we made needs.  I decided that there should be 1 block for every 12 squares, so the formula simply gets the number of squares in the grid, then divides by 12 and casts into an integer (which will round down by dropping any decimal points).

            int numberOfBlocks = (LightBlockEngine.maxX * LightBlockEngine.maxY) / 12;
            AddBlocks(numberOfBlocks);
        }

Now I add a function called "LocationValid" which will check a location sent in to make sure that it actually exists on the gameboard.  This will be useful for Movement commands later.

    public bool LocationValid(Location l) { return l.x >= 0 & l.y >= 0 & l.x < LightBlockEngine.maxX & l.y < LightBlockEngine.maxY; }

We then send that number to a new function in the grid class called "AddBlocks", which does a for loop based on the number of blocks that the board should have derived from the last function.

        private void AddBlocks(int numberOfBlocks)
        {
            for (int i = 0; i < numberOfBlocks; i++)
            {
                Square randomSquare = GetSquare();
                while (!FindValidBlockLocation(randomSquare.location))
                {
                    randomSquare = GetSquare();
                }
                board[randomSquare.location.x, randomSquare.location.y].currentState = SquareState.Block;
                ChangeState(randomSquare.location, SquareState.Block);
            }
        }

We use another function in the grid class called "GetSquare" which is overloaded.  If a location is sent as a parameter, then it simply returns a reference to that square in the board array.  If no parameters are sent, then it picks a square out of the board at random.

        public Square GetSquare(Location l) { return board[l.x, l.y]; }

        public Square GetSquare()
        {
            Random r = new Random();
            Location randomLoc = new Location(r.Next(0, LightBlockEngine.maxX), r.Next(0, LightBlockEngine.maxY));
            return GetSquare(randomLoc);
        }

I want to make sure that I actually get a valid square before I place a block.  So I put it in a while loop that gets random squares over and over until it finds a valid one using another function in Grid called "FindValidBlockLocation".  This makes sure that the square that was picked is set to the "Dark" state, then it uses the directions we made in Location earlier, while adding 4 more directions (North-west, North-East, South-West, South-East).  This checks each of the 8 squares surrounding the one we're checking to see how many of them are valid for the player to move at.  If there's 3 or more invalid locations around the square (including the edge of the gameboard), then it considers that location to be invalid for a block location.  The reason is, if it makes a cup shape where the player can only enter and then exit, then the puzzle might be impossible if there are two or more of those sorts of shapes.

        private bool FindValidBlockLocation(Location l)
        {
            if (GetSquare(l).currentState != SquareState.Dark)
                return false;

            List<Location> fullDirections = new List<Location>();
            fullDirections.AddRange(Location.Directions);
            fullDirections.AddRange(
                new Location[] { new Location(1, 1), new Location(1, -1), new Location(-1, 1), new Location(-1, -1) });

            int blocks = 0;
            foreach (Location direction in fullDirections)
            {
                if (!LocationValid(l + direction) ||
                    GetSquare(l + direction).currentState == SquareState.Block)
                    blocks++;
            }
            if (blocks >= 3)
                return false;

            return true;
        }

Returning to the "AddBlocks" function, if we got through that last function with a true, then the spot must be valid.  So let's change it to a block.

                board[randomSquare.location.x, randomSquare.location.y].currentState = SquareState.Block;
                ChangeState(randomSquare.location, SquareState.Block);

This calls the last function of the grid class "ChangeState" which is made to interact with the GUI.  I tried to make the GUI as separate from the mechanics as possible, so this is one of the very few places that calls to a GUI class in the mechanics systems.  The mechanics system doesn't need to know what the GUI class does, only to call it so the GUI can do it's own thing.

        public void ChangeState(Location l, SquareState s)
        {
            board[l.x, l.y].currentState = s;
            LightBlockEngine.gui.UpdateBlockColor(board[l.x, l.y]);
        }

Now we're done with the game board: I've got the grid constructed, the blocks added, and it can take variable grid sizes and change the number of blocks depending on that.

Now we need a Player class which will control player's location, movement, placement, and see if the player is stuck.  Let's start with a constructor.

        public Player()
        {
            this.location = PlacePlayer();
            LightBlockEngine.grid.ChangeState(location, SquareState.Player);
        }

This calls an overloaded function called "PlacePlayer" in the Player class.  Like "GetSquare", if it receives no parameter, it'll place randomly, if it receives a location, it'll place it there.   I don't have any particular place to put the player, so I use the random one.  It, like "FindValidBlockLocation", runs in a while loop until it finds a valid place to put the player.

        private Location PlacePlayer()
        {
            Square randomSquare = LightBlockEngine.grid.GetSquare();
            while (randomSquare.currentState != SquareState.Dark)
                randomSquare = LightBlockEngine.grid.GetSquare();
            return randomSquare.location;
        }

        public void PlacePlayer(Location l)
        {
            this.location = l;
            LightBlockEngine.grid.ChangeState(location, SquareState.Player);
        }

Now I need something to allow the player to move around the board.  I add a simple function called "Move", which checks to make sure that the location the player is trying to move is valid.  If it is, then it changes the state of the square the player is on to a normal "Light" state (wouldn't want to leave a trail of "Player" state squares when the player moves), then changes the square the player is moving to, to the player state.

        public void Move(Location l)
        {
            if (LightBlockEngine.grid.LocationValid(l) && 
                LightBlockEngine.grid.Board[l.x, l.y].currentState == SquareState.Dark)
            {
                LightBlockEngine.grid.ChangeState(location, SquareState.Light);
                this.location = l;
                LightBlockEngine.grid.ChangeState(l, SquareState.Player);
                CheckIfStuck(l);
            }
        }

Now that the player has moved, let's see if their new location has gotten them stuck and unable to move with the "CheckIfStuck" function.  This simply loops through the squares up, down, left, and to the right of the square the player is at.

        private void CheckIfStuck(Location l)
        {
            foreach (Location direction in Location.Directions)
            {
                if (LightBlockEngine.grid.LocationValid(l + direction) &&
                    LightBlockEngine.grid.GetSquare(l + direction).currentState == SquareState.Dark)
                    return;
            }

            if (CheckCompletion())
                LightBlockEngine.EndGame(true);
            else
                LightBlockEngine.EndGame(false);
        }

If none of them are valid movement locations, then the player is stuck and can move no more.  Now we need to figure out if that's because the player just lost or won.  I use a function called "CheckCompletion" which loops through every Square on the board and sees if any of them are dark.  If there are, then it returns false to say the player lost, if there aren't, then it returns true, the player won!  We make a call to the LightBlockEngine class to tell it the results.

Now that we've got a game board and a player who can move and win/lose the game, all we need is a class to tie it all together.  This is the "LightBlockEngine" class that we've been calling here and there.  This class will store references to all the classes the game needs to run: Grid, Player, and GUI.  

I start off with a static function in the class called "NewGame" which creates a new game (obviously).  This checks to make sure that the GUI reference exists so that things don't break, if it doesn't exist, then just return with an error.  Otherwise, the function takes in an x and y which dictates the grid size, and also a "size" which dictates the size (in pixels) of each square.  Using that, we create a new grid and player.

        public static void NewGame(int x, int y, int size)
        {
            if (gui == null)
            {
                Debug.WriteLine("ERROR! GUI not found!");
                return;
            }

            maxX = x;
            maxY = y;
            blockSize = size;
            grid = new Grid();
            player = new Player();
            origPlayerLoc = player.location;
        }

I store the player's original location as well so that we can add another function called "ResetGame", this will reset the game board as it was without randomizing the board again.  It simply sets the players location back to the "origPlayerLoc" location, and calls a function in the "Grid" class called "ResetGrid", which finds every square state that isn't set to "Block" and sets it to "Dark" (this erases the player square as well, but that's why we add the player back to it's original location afterwards).

And that does it for the mechanics!  The game is functionally complete, and set up to be as separate from the GUI as possible.  All it needs now is a GUI which will control all the visual aspects of the game, take keypresses for player movement, let the player choose the grid size, and communicate with the mechanics.  The windows form version of that GUI is linked to above under "GUI Code".

That's it!  It was a pretty fun project, and allowed some toying around with Object Oriented Programming structures.  Unfortunately, I'm not sure if every random puzzle it generates is possible, especially when the grid sizes start getting large, but a number of them should be.
