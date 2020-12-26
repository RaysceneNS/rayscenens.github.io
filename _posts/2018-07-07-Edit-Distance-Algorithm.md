---
title: "Levenshtein Edit Distance Algorithm "
tags: [Algorithms]
---

There are many algorithms in Computer Science that help us to understand our data in meaningful ways. Edit distance algorithms are one such set of tools.

## What is it

Edit distance algorithms are a mechanism that score how similar two strings are to each other. Given a source string (s) and target string (t), edit distance is computed as the minimum number of characters that must be inserted, deleted or substituted on the source string to morph it as the target string. An edit distance of zero indicates that the source and target strings are identical. An edit distance equal to the length of the longest string, would indicate that the strings are entirely dissimilar. For more information see this [Wikipedia Entry](https://en.wikipedia.org/wiki/Levenshtein_distance)

## The Scenario

Let's use a hypothetical scenario to illustrate how an Edit Distance algorithm can be useful.

1. Users in your Sales department create an application in Excel that records  a business deal. Values are input into cells of a spreadsheet to record products and where to send them.
1. The spreadsheet is shared around the department and becomes very popular.
1. At some point IT is called in to evaluate the spreadsheet and create a Single PageWeb  Application to replace it.
1. In the conversion process the existing data must be cleaned and transferred into a database.

Now here's where the challenge comes in, the users had been typing into plain text fields all the possible misspellings of place names. Our task is to match the misspelled names to known values.

We want 'Vancuover', 'Vancouve', 'Vancoiver', 'Cancouvers' -> 'Vancouver'

Using the Levenshtein  Edit Distance algorithm tells us the minimum number of keystrokes required to transform one string sequence into another.

So to edit the string 'Vancouve' -> 'Vancouver' requires us to insert 1 character.

This is an implementation of the edit distance algorithm written in c#.

```c#
using Microsoft.VisualStudio.TestTools.UnitTesting;
using System;

[TestClass]
public class UnitTest
{
    [DataTestMethod]
    [DataRow("Vancuover", "Vancouver", 2)]
    [DataRow("Vancouve", "Vancouver", 1)]
    [DataRow("Vancoiver", "Vancouver", 1)]
    [DataRow("Cancouvers", "Vancouver", 2)]
    [DataRow("", "Vancouver", 9)]
    [DataRow("Vancouver", "", 9)]
    [DataRow("", "", 0)]
    [DataRow("klasdi83jakjh", "Vancouver", 12)]
    [DataRow("van", "Vancouver", 7)]
    public void ExpectedCase(string s, string t, int expected)
    {
        var result = new LevenshteinEditDistance().Compute(s, t);
        Assert.AreEqual(expected, result, $"Calculate {s}, {t} expected {expected} but was {result}.");
    }

    [DataTestMethod]
    [DataRow(null, "")]
    [DataRow(null, "Vancouver")]
    [DataRow("", null)]
    [DataRow("Vancouver", null)]
    [ExpectedException(typeof(ArgumentNullException))]
    public void ExpectException(string s, string t)
    {
        new LevenshteinEditDistance().Compute(s, t);
    }
}

public class LevenshteinEditDistance
{
    public int Compute(string source, string target)
    {
        if (source == null)
            throw new ArgumentNullException(nameof(source));
        if (target == null)
            throw new ArgumentNullException(nameof(target));

        var sourceLength = source.Length;
        var targetLength = target.Length;

        var mtrx = new int[sourceLength + 1, targetLength + 1];
        for (var i = 0; i <= sourceLength; i++)
        {
            mtrx[i, 0] = i;
        }
        for (var j = 0; j <= targetLength; j++)
        {
            mtrx[0, j] = j;
        }

        for (var i = 1; i <= sourceLength; i++)
        {
            for (var j = 1; j <= targetLength; j++)
            {
                var cost = source[i - 1] == target[j - 1] ? 0 : 1;
                var del = mtrx[i - 1, j] + 1;
                var ins = mtrx[i, j - 1] + 1;
                var sub = mtrx[i - 1, j - 1] + cost;

                mtrx[i, j] = del > ins ? (ins > sub ? sub : ins) : (del > sub ? sub : del);
            }
        }
        return mtrx[sourceLength, targetLength];
    }
}
```
