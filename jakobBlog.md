# Jakob's Blog
 
 <!-- Nav -->
<table style="border:0px none;" width="100%">
	<tr>
		<td width="25%"> <a href="index.html"> Home </a> </td>
		<td width="25%"> <a href="amsBlog.html">Ashleys Blog</a> </td>
		<td width="25%"> <a href="jakobBlog.html">Jakob's Blog</a> </td>
		<td width="25%"> <a href="joshBlog.html">Josh's Blog</a> </td>
	</tr>
</table>

<!-- Main Content -->

## Automating a Boring Process for Our Designers
##### Date: 05/04/19

One of the first things I had taken into consideration regarding our rhythm game was the process of mapping the 'beats' (buttons that the player would tap) to a given audio file, a process I'll refer to as 'beatmapping'. We knew we would need some sort of visual interface or tool to assist, but struggled to think of ways to streamline the process outside of creating new software. Before the first week, another of our programmers (Ashley) had already a lot of progress on a custom Unreal Engine plug in, complete with an interface for visual assistance! 

Eventually, however, we settled on using a free to use software called Audacity.

```

#include "LoadAudacityLabelsToArray.h"
	
```

Upon loading an audio file into the Audacity software, we're shown a visual representation of the audio, known as a "waveform". Being able to 'see' our song brought convenience to the process of beatmapping, but how about actually adding the beats to the game? Initially we played around with the idea of manually filling an array in the engine itself, or using a graph in visual scripting (Blueprints) to mark floating point numbers.

Suddenly, a spark of genius! Audio isn't the only thing we can export with Audacity, it also supports the creation and exporting of labels! Labels were exported as a text file in a specific format, showing the start & end time of the label, as well as the description of the label.

<details><summary>Turning a text file into an array of strings.</summary>
<p>

```
TArray<FString> ULoadAudacityLabelsToArray::LoadTextFile()
{
	FString textfile = CreateFilePath();
	FString loadResult = ReadFileToString(textfile);
	PrintLoadResultSuccess(loadResult);
	TArray<FString> notes = SeparateTextFileByLine(loadResult);

	return notes;
}
```

</p>
</details>

<details><summary>Reading a text file with Unreal.</summary>
<p>

```python
FString ULoadAudacityLabelsToArray::ReadFileToString(FString &textfile)
{
	FString loadResult = "";
	UE_LOG(LogSpawnerPopulator, Log, TEXT("Attempting to read Audacity labels from file: %s"), *textfile);

	FFileHelper::LoadFileToString(OUT loadResult, *textfile);

	return loadResult;
}
```

</p>
</details>

The start time of the label would represent the 'perfect time' for the player to hit a note, while any notes the player has to 'hold' would also take advantage of the end time. The label description would specify which lane, type and special modifiers the note should spawn with. For example, "lpb" would spawn a note in the LEFT lane, with the POISON note type, with the BUBBLED modifier.

<details><summary>Reading a text file with Unreal.</summary>
<p>

```python
	// We have loadResult, a raw string version of the text file
	// Load result has the following structure:
	//  5.000000	5.000000    rlb
	//  7.000000	8.500000	lh
	//	(start_number) (tab) (end_number (for hold notes)) (tab) (label) ('/r' then '/n')
```

</p>
</details>

This allows our designers to 

Here's the function from the first working revision of the class that would actually do the interpretation, character by character. Be careful however, it contains a rather lengthy "if" statement.

<details><summary>if (reader == click here) {show mess}</summary>
<p>

```python
void ULoadAudacityLabelsToArray::InterrogateLines()
{
	// noteData.noteTime = GetLabelStart();
	// noteData.noteHoldLength = GetHoldTime(noteData.noteTime);

	// GetHoldTime() will include GetLabelEnd() and subtract it from GetLabelStart's value

	// noteData.bIsLeft = bGetNoteLane();
	// noteData.noteType = GetNoteType();
	// noteData.bIsBubbled = bGetNoteBubbled();

	FString decimalAndNumbers = "0123456789.";

	int32 lineNumber = 0;
	FString currentLine = "";
	FString currentChar = ""; 
	FString lastLetterSeen = "";
	bool bSpaceSeen = false;

	FString noteTimeString = "";
	FString noteTimeEndString = "";

	for (int32 i = 0; i < textFileLines.Num(); i++)
	{
		currentLine = textFileLines[i];
		lineNumber++;
		
		for (int32 j = 0; j < currentLine.Len(); j++)
		{
			currentChar = currentLine.Mid(j, 1).ToLower();

			if (decimalAndNumbers.Contains(currentChar))
			{
				if (bSpaceSeen) { noteTimeEndString.Append(currentChar); }
				else { noteTimeString.Append(currentChar); }
			}

			else if ((currentChar == " ") || (currentChar == "	")) { bSpaceSeen = true; }

			else if ((currentChar == "\n") || (currentChar == "\r")) 
			{ 
				if ((lastLetterSeen == "l") || (lastLetterSeen == "r")) { noteData.noteType = ENoteType::Default; }

				lastLetterSeen = "";
				
				bSpaceSeen = false;
			}

			else if (currentChar == bubbleLetter)
			{
				noteData.bIsBubbled = true;
				UE_LOG(GenerateNotesArray, Log, TEXT("Line number '%d': Just bubbled a note at %s seconds."), lineNumber, *noteTimeString);
			}

			else if ((lastLetterSeen == "l") || (lastLetterSeen == "r"))
			{
				if (currentChar == holdLetter)
				{
					noteData.noteType = ENoteType::Hold; 
					lastLetterSeen = holdLetter;

					noteData.noteHoldLength = FCString::Atof(*noteTimeEndString) - FCString::Atof(*noteTimeString);
				}

				else if (currentChar == twinnedLetter) // TODO should twinned note be a boolean? Twinned notes are intended to always be 'tap'
				{
					noteData.noteType = ENoteType::Twinned; 
					lastLetterSeen = twinnedLetter;
				}

				else if (currentChar == rightSwipeLetter)
				{
					noteData.noteType = ENoteType::Right_Swipe; 
					lastLetterSeen = rightSwipeLetter;
				}

				else if (currentChar == leftSwipeLetter)
				{
					noteData.noteType = ENoteType::Left_Swipe; 
					lastLetterSeen = leftSwipeLetter;
				}

				else if (currentChar == poisonLetter)
				{
					noteData.noteType = ENoteType::Poison; 
					lastLetterSeen = poisonLetter;
				}

				else if (currentChar == fakeLetter)
				{
					noteData.noteType = ENoteType::Fake; 
					lastLetterSeen = fakeLetter;
				}

				// TODO add additional note type checks here, like [ else if (currentChar == bombLetter) { noteData.noteType = ENoteType::Bomb; lastLetterSeen = "b"; }
				// Anything beyond this is error checking

				else if ((currentChar == "l") || (currentChar == "r"))	 { UE_LOG(GenerateNotesArray, Warning, TEXT("Line number '%d': Possible label naming error found in the text file at %s seconds. Please check the names of your labels."), lineNumber, *noteTimeString); }
				
				else if ((currentChar == " ") || (currentChar == "	")) { UE_LOG(GenerateNotesArray, Warning, TEXT("Line number '%d': Please don't put spaces in the label names."), lineNumber); }

				else { UE_LOG(GenerateNotesArray, Error, TEXT("Line number '%d': Unexpected character '%s' found in the text file at %s seconds. Please check the names of your labels."), lineNumber, *currentChar, *noteTimeString); }
			}

			else if ((currentChar == "l") || (currentChar == "r"))
			{
				if (lastLetterSeen == "") // Seeing an "l" or "r" without anything before it is the expected case
				{
					noteData.noteTime = FCString::Atof(*noteTimeString); // Set the time of the note
					
					if (currentChar == "l") { lastLetterSeen = "l"; bIsLeft = true; }
					else					{ lastLetterSeen = "r"; bIsLeft = false; }
				}
				else // the last letter seen is something unexpected, perhaps the label is 3 letters long or the designers tried to add a new note type beginning with "l" or "r"
				{ 
					UE_LOG(GenerateNotesArray, Error, TEXT("Line number '%d': Label naming error found in the text file at %s seconds. Please check the names of your labels."), lineNumber, *noteTimeString); 
				}
			}

			else  // Someone messed up the label name pretty badly to end up here.
			{ 
				lastLetterSeen = currentChar;

				UE_LOG(GenerateNotesArray, Error, TEXT("Line number '%d': Serious label naming error found in the text file at %s seconds. I saw a '%s'. Check the names of your labels."), lineNumber, *noteTimeString, *currentChar); 
			}
		}

		if (bSpamLogWithSuccessfulNotes) // this is just for logging and confirming the note type in real time
		{
			if (bIsLeft) 
			{ 
				leftSpawner->AddInputTime( noteData); 
				UE_LOG(GenerateNotesArray, Log, TEXT("Line number '%d': Note added successfully. Time: %s. Side: LEFT. Type: %s. Hold time: %s."), lineNumber, *noteTimeString, *EnumToString(noteData.noteType), *FString::SanitizeFloat(noteData.noteHoldLength));
			}
			else // is Right
			{ 
				rightSpawner->AddInputTime( noteData ); 
				UE_LOG(GenerateNotesArray, Log, TEXT("Line number '%d': Note added successfully. Time: %s. Side: RIGHT. Type: %s. Hold time: %s."), lineNumber, *noteTimeString, *EnumToString(noteData.noteType), *FString::SanitizeFloat(noteData.noteHoldLength));
			}
		}
		else
		{
			if (bIsLeft)  
				leftSpawner->AddInputTime( noteData ); 
			else		  
				rightSpawner->AddInputTime( noteData );
		}
		noteTimeString = "";
		noteTimeEndString = "";
		noteData.noteHoldLength = 0.0f;
		noteData.bIsBubbled = false;
	}
}
```

</p>
</details>

Wow.

As we can see in the last few lines, we communicate directly with the note spawner instances to use their AddInputTime functions. This adds the noteData of each individual note, including time, type, 'hold' length and "is bubbled".

I've recently 
