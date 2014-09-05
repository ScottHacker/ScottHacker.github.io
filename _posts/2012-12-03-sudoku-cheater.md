---
layout: post
title: "Sudoku Cheater in C#"
image: /images/sudoku_cheater.png
date: 2012-12-03
---

![Sudoku Board]({{ page.image }})

Sudoku is one of my favorite puzzle games, so I decided to make a project in C# based around seeing if I could completely solve a Sudoku puzzle automatically.  Currently it seems to be able to solve just about any Sudoku puzzle.

For those who haven't played Sudoku, the idea is: you have a 9x9 grid split into 9 boxes of 3x3.  Every row, column, and 3x3 box needs to have the numbers 1 to 9 without repeating any number.  A few numbers will be filled in already at start, and from that base, you should be able to solve any Sudoku puzzle completely through logic.

[Mechanics Code](https://github.com/ScottHacker/SudokuCheater/blob/master/SudokuCheater/Solve.cs) | [GUI Code](https://github.com/ScottHacker/SudokuCheater/blob/master/SudokuCheater/Form1.cs)

How I did it
--------------

I started off by creating the GUI that would take numbers in and store them in a 2-D byte array.  I used the type "byte" because I knew I would never need to store more than the number 9.  Once I have the actual array from the GUI, I send it to the Mechanics Code.  I wanted to keep the two as separate as possible so that I could change out the GUI easily without effecting any of the mechanics code.  

One of the tricks to solving Sudoku puzzles is the usage of pencil-marks: the idea is that you mark down the numbers a square could be in pencil, then when you find the right one, you write that in pen.  Normally you'd pencil-mark numbers when there's only 2-3 numbers a square could be to keep it from getting too chaotic, but this is a computer and it's going to be invisible to the user, so I'm going to pencil-mark every possible number for each square.

I use a function called "Markup" to do this, which is looped to repeat until it returns false, starting off by creating a Dictionary using a string and a List of bytes.  The string is going to be used for the key (which will be the grid x and y position together, so a square at position x: 1 and y: 2 will have the string "12"), and the list of bytes will keep track of the pencil marks.  Then I do two for loops so I can keep track of the grid position to use for keys in the Dictionary.  Once I get there, I check to make sure the space is empty (spaces with numbers in them are done, I don't need to worry about those).

{% highlight csharp %}
Dictionary<string, List<byte>> pencilMarks = new Dictionary<string, List<byte>>();

for (int x = 0; x < 9; x++)
{
	for (int y = 0; y < 9; y++)
	{
		if (puzzle[x, y] == 0)
		{
{% endhighlight %}


Now that I've got a loop finding each unsolved box, inside that loop I can start doing work.  I start off by making a copy of the puzzle array, because the first thing I'm going to do is try every possible number in it and find which ones don't work.

{% highlight csharp %}
			byte[,] tPuzzle = puzzle.Clone() as byte[,];
			for (byte i = 1; i < 10; i++)
			{
				tPuzzle[x, y] = i;
				if (!DuplicateCheck(GetBox(x, y, tPuzzle)) & !DuplicateCheck(GetColumn(y, tPuzzle)) & !DuplicateCheck(GetRow(x, tPuzzle)))
				{
					if (pencilMarks.ContainsKey(x.ToString() + y.ToString()))
						pencilMarks[x.ToString() + y.ToString()].Add(i);
					else
						pencilMarks.Add(x.ToString() + y.ToString(), new List<byte> { i });
				}
			}
		}
	}
}
{% endhighlight %}


To do that, I need to take each square and find the row, column, and box connected to that so I can compare their numbers.  These are the functions: GetBox, GetRow, and GetColumn.

{% highlight csharp %}
private byte[] GetRow(int x, byte[,] puzzle)
{
	byte[] n = new byte[9];
	for (int i = 0; i < 9; i++)
	{
		n[i] = puzzle[x, i];
	}
	return n;
}



private byte[] GetColumn(int y, byte[,] puzzle)
{
	byte[] n = new byte[9];
	for (int i = 0; i < 9; i++)
	{
		n[i] = puzzle[i, y];
	}
	return n;
}

private byte[] GetBox(int x, int y, byte[,] puzzle)
{
	List<byte> n = new List<byte>();

	int xQuadMax = GetMaxRange(x);
	int yQuadMax = GetMaxRange(y);

	for (int i = xQuadMax - 3; i < xQuadMax; i++)
	{
		for (int j = yQuadMax - 3; j < yQuadMax; j++)
		{
			n.Add(puzzle[i, j]);
		}
	}

	return n.ToArray();
}
{% endhighlight %}


Once I get those numbers from the rows, columns, and boxes, I plug them into a function called DuplicateCheck.  This function creates a list of bytes, so I can use some built-in functionality from lists that checks for duplicates.  Afterwards, I go through each number in the list and check to see if it exists in our list, if not, then we add it.  If we find a duplicate, then it returns true immediately, if it doesn't, then it returns false.

{% highlight csharp %}
private bool DuplicateCheck(byte[] numList)
{
	List<byte> nums = new List<byte>();
	foreach (byte n in numList)
	{
		if (n != 0)
		{
			if (nums.Contains(n))
				return true;
			nums.Add(n);
		}
	}
	return false;
}
{% endhighlight %}


If we didn't find any duplicates in the rows, columns, or boxes, then we add the pencilmark to the dictionary.  I need to check to make sure that the key exists or not first so I can either create a new entry in the dictionary or add to an existing one.

{% highlight csharp %}
if (pencilMarks.ContainsKey(x.ToString() + y.ToString()))
	pencilMarks[x.ToString() + y.ToString()].Add(i);
else
	pencilMarks.Add(x.ToString() + y.ToString(), new List<byte> { i });
{% endhighlight %}


Now that we've got a Dictionary with pencil-marks for each square completed, we have a base to do some logic from.  We're done with the big loop, but not with the "Markup" function.  Next thing is to do some logic to adjust the pencilmarks a bit.  We start off with the "FindDuplicatePencilMarks" function.  This modifies the pencil marks alone instead of trying to set a number, so it'll be useful for the other logic functions to figure this one out first.

{% highlight csharp %}
FindDuplicatePencilMarks(ref pencilMarks);
{% endhighlight %}


We create a list of Intersections, which is a simple struct I created for this purpose (detailed below).  All it does is hold the keys for two squares and the number combo they have in common.  

{% highlight csharp %}
struct Intersection
{
	public string key1;
	public string key2;
	public List<byte> combo;

	public Intersection(string k1, string k2, List<byte> c)
	{
		key1 = k1;
		key2 = k2;
		combo = c;
	}
}
{% endhighlight %}


Once we do that, we have to go through each Row, Column, and box, of pencil marks.  Since this is returning pencil marks, not penned in numbers, I can't use the normal GetColumn, GetRow, and GetBox functions I wrote above, so I overload them with a new function that does the exact same thing but gets pencil marks instead.  Now I plug that into the parameters of a new function called "DuplicatePMCheck", which returns an Intersection to be added to the list.

{% highlight csharp %}
private void FindDuplicatePencilMarks(ref Dictionary<string, List<byte>> pencilmarks)
{
	List<Intersection> results = new List<Intersection>();

	foreach (string key in pencilmarks.Keys)
	{
		results.Add(DuplicatePMCheck(key, GetRow(int.Parse(key[0].ToString()), pencilmarks)));
		results.Add(DuplicatePMCheck(key, GetColumn(int.Parse(key[1].ToString()), pencilmarks)));
		results.Add(DuplicatePMCheck(key, GetBox(int.Parse(key[0].ToString()), int.Parse(key[1].ToString()), pencilmarks)));
	}
{% endhighlight %}


Moving into the function "DuplicatePMCheck", we do all the actual logic for figuring out whether or not a Column, Row, or box has the duplicate pencil marks we want.  

The logic behind this, in terms of Sudoku, is that if you see a row with two squares in that row that both have, for example: a 2 and a 7.  If no other square in that row has a 2 or a 7, then those squares MUST be either a 2 or a 7 and nothing else, so we can disregard any other pencil marks there.

The first thing to do with this function is get the list of pencil marks for the box we're checking against.  We take that list, and then find every combination of two of those pencil marks.  So if the box with key "12" has pencil marks: 1, 2, & 3, then we find combos: 1 & 2, 1 & 3, and 2 & 3.  I do this by looping through each number twice to compare against each other, then ignore the numbers that are equal (so we don't get a match of 1 & 1), once I've done that, I add the number to a list with both combo potentials in it and sort it (to make sure I don't get 1 & 2 and 2 & 1 being treated as different combos).  Now I use LINQ to run through the list of lists to see if the combo exists or not in there already, if it doesn't, I add it.

{% highlight csharp %}
private Intersection DuplicatePMCheck(string oKey, Dictionary<string, List<byte>> numLists)
{
	List<List<byte>> combinations = new List<List<byte>>();

	foreach (byte n in numLists[oKey])
	{
		foreach (byte i in numLists[oKey])
		{
			List<byte> combo = new List<byte>() { n };
			if (i != n)
			{

				combo.Add(i);
				combo.Sort();

				if (!combinations.Any(o => o.SequenceEqual(combo)))
					combinations.Add(combo);
			}
		}
	}
{% endhighlight %}


Now that we've got a list of every valid two number combination of the pencil marks, we need to start parsing through the Row, Column, or Box's pencil marks to look for duplicates.  This goes through each combo, then each pencil mark list to check.  First thing it does is make sure to exclude the original square we're checking against from this check, we wouldn't want the square to match with itself.  

{% highlight csharp %}
foreach (List<byte> combo in combinations)
{
	List<Intersection> intersections = new List<Intersection>();

	foreach (string key in numLists.Keys)
	{
		if (oKey != key)
		{
{% endhighlight %}


Then it does an exclusive or (XOR) gate to check if either number in the combo exists in a square without the other.  If only one number of that combo exists in any square, than the whole combo is invalid, so we clear the list and then break out of the loop so that it'll continue to the next combo.  Then it checks if both numbers of the combo are contained in a square, if so then we add it to the list and move on.

{% highlight csharp %}
		if (numLists[key].Contains(combo[0]) ^ numLists[key].Contains(combo[1]))
		{
			intersections.Clear();
			break;
		}

		if (numLists[key].Contains(combo[0]) && numLists[key].Contains(combo[1]))
		{
			intersections.Add(new Intersection(oKey, key, combo));
		}
	}
}
{% endhighlight %}


Once we're done with that particular combo in the combinations loop, we check to see how many hits there were.  We don't want to have a scenario where 2 & 7 exist more than 2 times in a row, column, or box being returned as true.  If there's only one item in the intersections list, then we know that we've got a hit, so return that.  We don't even need to go through the rest of the combinations loop.  Otherwise, if we go through the entire combinations loop with nothing getting returned, then we return a blank Intersection that will be used as a null type.

{% highlight csharp %}
	if (intersections.Count() == 1)
		return intersections[0];
}

return new Intersection();
{% endhighlight %}


Now we go back to the "FindDuplicatePencilMarks" function.  We'll have ran "DuplicatePMCheck" on every Row, Column, and Box related to the square we're checking against now and have the results in the form of a list of Intersections.  All we have to do now is run through that list and apply the results.  So I loop through the list, and check to make sure that the combo isn't null (if it is, then it's the default Intersection type that I used as a null), if not, then I get both the keys for squares from the Intersection struct and set their pencil marks to equal the combo alone.

{% highlight csharp %}
foreach (Intersection i in results)
{
	if (i.combo != null)
	{
		pencilmarks[i.key1] = i.combo;
		pencilmarks[i.key2] = i.combo;
	}
}
{% endhighlight %}


Now that we're done with that and back in the "Markup" function, I use the pencil marks to do all the logic to figure out numbers.  All these functions will return true if they find anything, and false if they don't.  This will become our return value, since if the function couldn't find any new numbers, then it effectively can't solve the puzzle at all.  If it can, then it'll repeat the function with the new numbers in the hopes of finding more.

{% highlight csharp %}
return FindSingles(ref puzzle, pencilMarks) | FindNegativeSingles(ref puzzle, pencilMarks);
{% endhighlight %}


Next is the "FindSingles" function.  This, very simply, looks through the pencil-marks and see if any of them have only one pencil-mark in their array.  If that's the case, then that pencil-mark is obviously the correct number for that spot, so we pen it in the official puzzle array and return true.

{% highlight csharp %}
private bool FindSingles(ref byte[,] puzzle, Dictionary<string, List<byte>> pencilMarks)
{
	bool changed = false;
	foreach (string key in pencilMarks.Keys)
	{
		if (pencilMarks[key].Count == 1)
		{
			puzzle[int.Parse(key[0].ToString()), int.Parse(key[1].ToString())] = pencilMarks[key][0];
			changed = true;
		}
	}
	return changed;
}
{% endhighlight %}


The second logic function is "FindNegativeSingles", this cross-references the rows, columns, and boxes and looks for the places a number can't be in one of those areas.  It's possible that a square could have multiple pencil-marks but there's only one place a number could be through process of elimination.  I do this by going through every square on the grid, then checking their rows, columns, and boxes and use a function called "UniqueNumberCheck" to see if any of the pencil-marks is unique in their respective group.  That function will either return a 0 if it can't find anything, or the unique number if it can.  

So I use Math.Max to go through the results, if one of the returns is a unique number, it'll set the result to that, otherwise it'll set it to 0.  Once I've got the result, I check to see if it's 0 or not, if it isn't, then I update the grid with the unique number and return true to say that the grid has been changed.

{% highlight csharp %}
private bool FindNegativeSingles(ref byte[,] puzzle, Dictionary<string, List<byte>> pencilMarks)
{
	bool changed = false;
	foreach (string key in pencilMarks.Keys)
	{
		byte uniqueNumber = Math.Max(Math.Max(
			UniqueNumberCheck(pencilMarks[key], GetRow(int.Parse(key[0].ToString()), pencilMarks)),
			UniqueNumberCheck(pencilMarks[key], GetColumn(int.Parse(key[1].ToString()), pencilMarks))),
			UniqueNumberCheck(pencilMarks[key], GetBox(int.Parse(key[0].ToString()), int.Parse(key[1].ToString()), pencilMarks))
				);
		
		if (uniqueNumber != 0)
		{
			puzzle[int.Parse(key[0].ToString()), int.Parse(key[1].ToString())] = uniqueNumber;
			changed = true;
		}
	}
	return changed;
}
{% endhighlight %}

And that's pretty much it!  Extra logic functions can easily be added if more is necessary to solve more difficult puzzles.
