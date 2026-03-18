# Word Boggle Overview

## Word Hashing
Word Hashing is done and managed by ```WordsManager``` class and remain persistant and non-singleton across the game scenes.
Its accessibility is done by an ```EventManager``` class (More on it in future)

Since we have 55681 words to work with, I decided to go with a Tree. This is because of the following reasons:
1. Searching a word in a Tree has less complexity ```O(word.length)``` rather than ```O(totalWords)``` in an Array/List.
2. Trees are highly versatile when it comes to their customization. (For Example : Adding a new word will reuse the existing nodes)

The structure of a Tree node has a following format:

```c#
    public class Node
    {
        public char le; // letter
        public bool isValidWord = false; // Is current Word valid
        public readonly Dictionary<char , Node> children = new(); // reference to children nodes
    }
```
<img width="863" height="666" alt="Screenshot 2025-08-10 161902" src="https://github.com/user-attachments/assets/58816b4e-20fd-4256-9081-14d94c6d119c" />

I have also added a callback for the "OnWordsLoaded" upon which we can make the interactions enabled or disable an ongoing loading screen. Though it is currently being used for enabling interactions only.


### Addons :
Since we have words divided among 6 types of length ```(3,4,5,6,7,8)```. In order to introduce difficulty and more control we can create trees based upon the length and traverse them based on the probability of occuring a certain length of word.
For example : 50% Prob of getting a 6 letter word, Therefore we can use algorithm of ```Weight Based Probability Selection``` which will yield number 6 with 50-50 chance, Proceeding with it, we can retrieve a word from a tree which only contains 6 letter words.

## Event-Management

I wanted to create this game such that there are no singletons and every class will be responsible for itself only. Therefore I have created a static class ```EventManager``` which will be containing all the statically defined public events and can be access throughout the game. Upon triggering a specific situation rather than storing references to all the classes require to act on, I am just invoking a specific Action / Func which will provide me the logic execution without any extra reference management. 

This type of architecture follows <b>Observer Design Pattern</b> which I use in almost every project.

## Grid Handling

The grid handling is done ```GridHandler``` class and uses ```WordsManager``` class and is core of the game.

Here I am inserting the words in such a way that a new word can also use the letters of past words embedded. Also it can detect whether a similar is already being made or not.
For example : Word "Clear" also contains word "ear"


For fast retrivals I have maintained a Letter-Registry ```LetterReg``` which is used to retrive the grid indices with existing letters in it. For Example : I don't have to iterate over the grid just to find a tile containing the required letter to be reused.

One of the most crucial and best thing about this script is that... it is using "Collection Pooling". For Example, consider the following function : 
```c#
    private bool InitiateWordAddition(ref string str, bool isFirst = false)
    {
       // bool res = false;
        List<LetterTile> tiles = CollectionPool<List<LetterTile>, LetterTile>.Get();
        LetterTile tile = null;
        int index = 1;

        //Try Getting the tiles from every similar existing letter
        if (LetterReg.ContainsKey(str[0]))
        {
            foreach (var i in LetterReg[str[0]])
            {
                Debug.Log(i);
                tile = grid[(int)i.x, (int)i.y];
                tiles.Add(tile);
                index = 1;
                if (TryAddWord(ref tiles, ref str, ref tile, ref index, true))
                {
                    UpdateLetterRegistry(ref tiles);
                    CollectionPool<List<LetterTile>, LetterTile>.Release(tiles);
                    Debug.Log($"Word {str} added with existing scan");
                    return true;
                }

                foreach (var t in tiles)
                {
                    if (i == t.GetTileIndex())
                    {
                        continue;
                    }
                    t.SetLetter(string.Empty);
                }

                tiles.Clear();
            }

        }

        // FALLBACK : Try additing the word from every valid tile
        return TryAddWordWithEveryValidTile(ref str, ref tiles, isFirst);
    }
```

This function is responsible for the addition of a word in the grid.

```c#
List<LetterTile> tiles = CollectionPool<List<LetterTile>, LetterTile>.Get();
```
provides me a reference to the List<LetterTile> which was created in past and now cleared and ready to be reused.
If there wasn't any such collection created in past, then Unity will create anew and provide it.

Once done with our work we can release that resource, so that it can be reused in the future: 

```c#
if (TryAddWord(ref tiles, ref str, ref tile, ref index, true))
{
    UpdateLetterRegistry(ref tiles);
    CollectionPool<List<LetterTile>, LetterTile>.Release(tiles);
    Debug.Log($"Word {str} added with existing scan");
    return true;
}
```
### Searching a word

In gameplay if word does not exist inside a list of embedded words, then it will fallback to searching that word in the Tree itself in ```WordsManager```
if the required word gets found in any of the searching, then it will add that word into a list of existing words in order to avoid the any kind of exploitation.


## Game Stats
In order to manage the game stats, I have created a mini architecture inspired from Model-View-Controller. Though it is not completing that.
Here the functionality is being divided based on the kind of work required to be done.

```StatsController.cs``` is responsible for the logic and behaviour of the stats.<br>
```StatsView.cs``` is responsible to reflect that data on UI.

## Word Validation
The word validation is done by ```GridHandler``` class due, however the logic initiates executiing by ```BoggleHandler``` class.
For registering and keeping track of the words made I am using ```IPointerHandler``` interfaces which are providing callbacks based on the tap and drag. 
In order to check a word determining it via an Enum ```WordValidationType``` and invoking the corrosponding event based on the verdict.

```c#
switch (EventManager.OnValidateWord.Invoke(str))
{
    case WordValidationType.VALID:
        OnValidWord(tiles);
        break;
    case WordValidationType.INVALID:
        OnInvalidWord();
        break;
    case WordValidationType.EXISTING:
        OnExistingWord();
        break;
    default:
        Debug.LogError("Unknown validation verdict");
        break;
}
```

## Letter Tile

In order to modify the data and visual of a Letter tile in one call, I decided to wrap the calls of the data change in the properties itself, providing me relief from adding another event in EventManager class or any explicit calls

```c#
    private bool m_IsBonus = false;
    private bool m_IsBlocked = false;
    public bool IsBonus
    {
        get => m_IsBonus;
        set
        {
            SetAsBonusTile(value);
        }
    }
    public bool IsBlocked
    {
        get => m_IsBlocked;
        set
        { 
            SetAsBlockedTile(value);
        }
    }

    private bool m_isSelected = false;
    public bool IsSelected
    {
        get => m_isSelected;
        set
        {   
            m_isSelected = value;
            OnSelected(value);
        }
    }
```

## Pre-Defined Data

I am storing the predefined data in two places: 

For the data like which gets used in the core of the game, is being stored in ```Constants.cs``` <br>
For the data whose values can be customized eg Scoring and etc are being stored in a ```Scriptable Object``` called ```GameConfig```.

## Optimizations Considered
1. Collection Pooling
2. Keep param passing in tight loops as "pass-by-reference"
3. Event-driven communication
4. Avoided Update Functions as much as possible.

## Corner Cases tackled 

1. While Adding the words in the grid, it is possible to repeat on same tiles for word addition resulting in incomplete / wrong word addition 

```c#        
    // Get the neighbours
    neighbours = GetNeighbours(startTile.GetTileIndex());

    // CORNER CASE : words like "fgf" can redirect again to one of the accquired tiles in the past calls. Therefore removing them from the list of neighbours
    foreach (var lastTiles in tiles)
    {
        neighbours.Remove(lastTiles);
    }
```

2. User can exploit the grid by dragging the finger out of the grid and pointing it to a farther cell. In order to fix this I added a boundary checks in the borders, which triggers the "End Word Selection".

<img width="788" height="842" alt="Screenshot 2025-08-10 161832" src="https://github.com/user-attachments/assets/5dc13852-3773-4703-9f35-6e6c36008080" />

