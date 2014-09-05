---
layout: post
title: "Numbers-to-Words in F#"
image: /images/FunctionalNumbersToWords.PNG
date: 2013-01-22
---

![Numbers to Words in F# interactive]({{ page.image }})

I've been looking into more Functional Programming lately, having heard about F#, Microsoft's relatively new Functional programming language.  I decided to challenge myself in picking it up and converting my earlier code into as pure functional programming as I could.  I decided to go with converting Numbers To Words to simplify the GUI process.  

It was quite a bit trickier than I expected, but it was not just using a different language but a different coding paradigm.  Not being able to use mutable variables made me recognize a lot of things I took for granted, but I think in the end it made for more efficient code with less side-effects.

The Idea behind the program, as before, is to take a non-negative integer and convert it into a properly formatted English sentence.  For more details on the basic program read [Numbers to Words](https://github.com/ScottHacker/Number-to-words) as this will largely be concerned with converting it from C# to F#

[Source](https://github.com/ScottHacker/Number-to-words_Functional/blob/master/NumbersToWords.fs)

How I did it
-------------

The first thing to do was to convert the NumberToWord function, which managed the work of converting integers from 1-100 into words.  In the previous C# function, I used a case statement with a few tricks to keep the length of the whole thing down.  In this one, trying to avoid mutable strings, I decided to keep it more simple.  F# also has a much nicer match system which lets me match ranges as well as direct values that helped a lot.  In the end, even though this took longer to write, I think it looks neater and less confusing than the previous one.

    let NumberToWord n =
        match n with
            | 1 -> "One"
            | 2 -> "Two"
            | 3 -> "Three"
            | 4 -> "Four"
            | 5 -> "Five"
            | 6 -> "Six"
            | 7 -> "Seven"
            | 8 -> "Eight"
            | 9 -> "Nine"
            | 10 -> "Ten"
            | 11 -> "Eleven"
            | 12 -> "Twelve"
            | 13 -> "Thirteen"
            | 14 -> "Fourteen"
            | 15 -> "Fifteen"
            | 16 -> "Sixteen"
            | 17 -> "Seventeen"
            | 18 -> "Eighteen"
            | 19 -> "Nineteen"
            | 20 -> "Twenty"
            | _ when n > 20 && n < 30 -> "Twenty-"
            | 30 -> "Thirty"
            | _ when n > 30 && n < 40 -> "Thirty-"
            | 40 -> "Fourty"
            | _ when n > 40 && n < 50 -> "Fourty-"
            | 50 -> "Fifty"
            | _ when n > 50 && n < 60 -> "Fifty-"
            | 60 -> "Sixty"
            | _ when n > 60 && n < 70 -> "Sixty-"
            | 70 -> "Seventy"
            | _ when n > 70 && n < 80 -> "Seventy-"
            | 80 -> "Eighty"
            | _ when n > 80 && n < 90 -> "Eighty-"
            | 90 -> "Ninety"
            | _ when n > 90 && n < 100 -> "Ninety-"
            | _ -> null

Next is to convert the function that gets the "length" of an integer.  The original one used a while loop, but loops are pretty much out due to mutable values, so I use a recursive function.  This will keep dividing by 10 and calling itself until the results are under 10, at which point it'll add all the numbers on the stack and return the total.

    let rec GetLength n =
        let length = 1;
        match n with
        | _ when n >= 10 -> length + GetLength (n/10)
        | _ -> length

Once we have that, I can make a function that will take a number and split it into groups of threes.  This is another recursive function, since the original version of this required a for loop to grab them all.  I use the basic Math libraries to do the Power of math instead of making my own to save a bit of space, and use that to get each group.  The format that this returns is one of the other nice things about F#, as it uses Tuples, which are basically a list of various types (for example: one tuple may be (string, int) ).  I use tuples here so I can keep track of which number the group is in a list for later use. So each run of this function will prepend a tuple to a list then return the list when done.  For example, plugging in 123456789 will return [(123, 2); (456, 1); (789, 0)]

    let rec GetGroups n =
        let length = GetLength n
        let e = int(float(length) / 3.0 - 0.1)
        let power = int(System.Math.Pow(1000.0, float(e)))
        let group = n / power
        match length with
        | _ when length > 3 -> (group, e) :: GetGroups (n - group * power)
        | _ -> [(group, 0)]

Now that I have a function to turn any function that will turn any number from 1-100 into a word and a function that will break a big number into groups of threes, I can start doing the real work.  Unlike the C# version, I decided to try and keep the string building to one function as much as I could, then do one print line of the results at the end.  This function will build the words only, I'll do punctuation in a later function.  

In the process, I also discovered an easier way to find the variable "i", by using the length of the 3 number group minus 1 as an exponent of 10.  For example, sending in the number 12 would get me the length 2, minus 1 gets 1, so the math is 10 ^ 1 which equals 10.  Whereas a length of 3 would get 3 - 1 = 2, then 10 ^ 2 = 100.  This allows me to find the number I need entirely with a mathematical function instead of a clumsy 3 part if statement.

After that, I make a local function called AddSize within GetWords.  This is very simple, so I kept it to one line.  Remember the second part of the tuple returned from GetGroups?  Now we use that.  This takes that integer (cast as "g" in this function) which marks this particular group's position in the list, and matches it to a string which will define the size of the number group, i.e.: Billion, Million, Thousand.

Now the final part, which matches the variable "i" to the appropriate number.  If it's 100, then it gets the first number from the NumberToWord function, adds "Hundred", then checks to see if there's more in the number group by hitting it with modulus 100.  If there is, then it adds "and" and recursively calls itself to get the rest, otherwise it runs the "AddSize" function to add the group and return.  When the function recursively calls, it'd either hit the 10 or the 1.  If it hits the 10, then it'll run NumberToWord then check to see if it's over twenty, if it is, then it runs NumberToWord to get the rest of the word, in the case of words like "Twenty-Six" or "Thirty-Four".  Once it does, it runs "AddSize" and automatically returns.  If it hits 1, then it checks the "g" variable to see if it's the last group in the number, if it isn't, then it gets NumberToWord then runs AddSize, otherwise it just runs NumberToWord.

Once that's done, all we have to do is return the list.  If we plugged (12, 1) into this, then it would return ["Twelve"; " Thousand"], whereas putting in (345, 0) would return ["Three"; " Hundred"; " and "; "Fourty-Five"; ""].  The extra empty string is due to the "_" in AddSize, which is a wildcard for all values that the ones I supplied didn't cover.  It will just come up as an empty string here, so it won't cause us any problems when we put it all together.

        let rec GetWords (n, g) =
            let i = int(System.Math.Pow(10.0, float(GetLength n - 1)))
            let words = []
            let AddSize = match g with | 3 -> " Billion" | 2 -> " Million" | 1 -> " Thousand" | _ -> ""
            let words = match i with
                        | 100 -> words @ [NumberToWord(n/i); " Hundred"] @ if n % i > 0 then [" and "] @ GetWords((n%i), g) else [AddSize]
                        | 10 -> words @ [NumberToWord n + (if n > 20 then NumberToWord (n%i) else ""); AddSize]
                        | 1 -> words @ if g = 0 then [NumberToWord n] else [NumberToWord n; AddSize]
                        | _ -> words
            words

Now for the final and main function that glues it all together with proper punctuation.  I named this one "Say" so that typing it into Visual Studio's F# interactive would come out as something like "Say 12345".

The first thing it build the raw word groups into a List of List of Strings.  For example, 12345 would come out as [["Twelve"; " Thousand"]; ["Three"; " Hundred"; " and "; "Fourty-Five"; ""]].  The separation of the word groups here is useful, since I can use that to put in punctuation later.  As before, I also check to see if the user has put in "Say 0" so I can just skip the whole thing and return "Zero".

Once we have that, the next goal is to go through the list of lists of strings and glue them together into a single list of strings with proper punctuation in place.  It does this by making a recursive local function called "buildList", which gets the first element of the list using "groups.Head".  It starts off by adding the head to the wordList then checks if the Tail (representing everything [i]but[/i] the head of the list) is empty or if the first item of the first list of the tail is null, if it is, then it adds a punctuation mark and returns.  If not, then it checks to see if the length of the rest of the tail is only 1 list, and if that list is only 1 item long.  If it is, then we've got a situation like "5001" which needs different formatting than "5101". otherwise we add a comma then run the function again to move to the next list.

Now that we're done with that, we have a single dimensional list with all the words and punctuation.  All we have to do now is convert it to string.  I do this using a simple local function called "buildSentence".  This, like the last function, adds the first word of the list to the string, then checks if there's anymore, if it is, then it recursively calls itself to continue down the rest of the list, otherwise it adds an empty string and returns.

Once that's done, all we have to do is print the string with a line break and it's finished!

        let Say n =
            let wordGroups = match n with
                                | _ when n > 0 -> List.map GetWords (GetGroups n)
                                | _ -> [["Zero"]]
        
            let wordList = []
            let rec buildList (groups:string list list) =
                wordList @ groups.Head @ 
                        if groups.Tail.Length = 0 || groups.Tail.Head.Head = null then ["."] 
                        else 
                            (if groups.Tail.Length = 1 && groups.Tail.Head.Length = 1 then [" and "] else [", "]) @ (buildList groups.Tail)
            let wordList = buildList wordGroups
        
            let sentence = ""
            let rec buildSentence (words: string list) =
                sentence + words.Head + if words.Tail.Length > 0 then (buildSentence words.Tail) else ""
            printf "%sn" (buildSentence wordList)
