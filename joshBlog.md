# Josh's Blog
 
 <!-- Nav -->
<table style="border:0px none;" width="100%">
	<tr>
		<td width="25%"> <a href="index.html"> Home </a> </td>
		<td width="25%"> <a href="amsBlog.html">Ashleys Blog</a> </td>
		<td width="25%"> Jakobs Blog </td>
		<td width="25%"> Jhoes Blog </td>
	</tr>
</table>

<!-- Main Content -->

## Audio 2.0
##### Date: 24/03/19

Helloo all! I'm Ashley and I’m 1 of 3 programmers at OverScope Studios who is currently working on Jazz Odyssey an audio rhythmic targeted at the mobile market. I am also currently a first-year student at Falmouth university studying Computing for Games.

This is the first of (hopefully) many blog posts regarding the issues I have faced with game development and how I have overcome the problem. In this blog post we will be looking at the audio system behind Jazz Odyssey

A couple of weeks ago during a team meeting one of the designers mentioned that there is no background music during the dialogue phases of Jazz Odyssey. To be fair I hadn’t really given it much thought since alpha stage as we had so much to implement, but it has all calmed down now and we can mainly focus on refactoring and improvements to our code base. So I decided to take it on and implement the dialogue background audio and the other audio features that are currently missing such as sound effects (i.e. filters, pitch change and fading etc...). But it turns out that it was not going to be as simple as just refactoring the code base but required a total rewrite of the audio systems.

When I originally started working on the audio system I ran into a number of problems such as timing issues where inputs would fall out of sync, the TimePrecentage call back in unreal would not work on android and crashes if there were no audio devices initialized just to name a few. This took a fair bit of research to overcome the issues since I was new to the Unreal engine and as a result it lead to a rather messy code base that had been "hacked" together through trial and error.

After an hour or so of trying to figure out how to refactor the code it became apparent that I needed to redesign the entire system from the ground up, since there where unused functions being called within the audio manager itself, bad naming conventions and an over complicated code. For instance, there was one function that really stood out as being badly named and over complicated called SetAudioPosition that was something like

```
Procedure SetAudioPosiution(delta time : float):

	if audio has not started:
		return
	
	if audio time > audio duration:
		ended = true
		
	if ended and audio time < audio duration + end time:
		audio time += delta time
		
	if not ended:
		audio time += delta time
		
End procedure
	
```
For a start it was called in Tick and none of the variables where used, we actually used a variable called audioFinsihed instead of ended that was set in OnAudioFinished and the audio time came from a function called GetAudioPosition, so the SetAudioPosition function itself was totally useless.

So once I had decided that I should redesign the system from the ground up, I first thought about what functions should stay and be refactored and what should be removed. It turned out I only needed to refactor GetAudioPosition since it was the main time system used in the game to keep everything in sync with the audio, and it was used throughout the entire code base, everything else however could be deleted safely even playAudio. The next thing I had to do was decide how to implement the new audio time system, so first I added a function called UpdateAudioDelta which simple gets the amount of time that has past since the last audio update.

```
void AudioManager::UpdateAudioDelta(){

	float audioTime = UGameplayStatics::GetAudioTimeSeconds(world);	// World audio time

	current_delta = audioTime - last_deltaUpdate;
	last_deltaUpdate = audioTime;
	
}
```

I then added a function to update the position of the of the audio called UpdateAudioPosition


```
void AudioManager::UpdateAudioPosition(float delta){

	if(!audioStarted) return;	// return if there no audio playing
	
	currentAudioPosition += delta;

}
```

I then added 2 getters to the AudioManager so that the AudioDelta and AudioPosition where both available publicly in the code. This now meant that the game was working again although there was no audio playing in the game. So, the next decision I had to make was how can I implement the music to both the dialogue and main game and decided that I could use just a single audio component since there would be a second or two between the dialogue music and the main game music. So, for that I added a Sigle audio component and next audio clip variables to the header file.

```
AudioManager.h

private:
	UAudioComponent audioComp;
	USoundWave nextAudioClip;
	
```

followed by adding functions to set and trigger the next audio clip

```
AudioManager.cpp

void AudioManager::SetNextAudioClip(USoundWave* audio, bool loop)
{

	nextAudioClip = audio;

	if (audio)	// check that there is audio to be looped
		nextAudioClip->bLooping = loop;

}

void AudioManager::PlayNextAudioClip(float audioPosition)
{

	if ( !nextAudioClip ) return;	// no audio to play
	

	// set the current audio position
	if ( audioPosition < 0 ) audioPosition = 0;
	currentAudioPosition = audioPosition;

	// update the audio component with the next track
	audioComp->Sound = nextAudioClip;
	nextAudioClip = nullptr;

	float startVolume = volume;

	audioComp->SetVolumeMultiplier(startVolume);
	audioComp->Play(audioPosition);
	audioStarted = true;
}

```

Next I needed to add a method to fade the audio in and out, so to achieve this I created a new enum called AudioFadeType with 'None', 'In' and 'Out' to track witch fade state we are currently in then implemented a function called FadeAudio

```
void AudioManager::FadeAudio(delta)
{
	// find the volume by the position of the fade.
	float volumeMultiplier = 1.0f;

	currentFadeTime += delta;

	// prevent the volume going over 100%
	if (currentFadeTime > fadeTime) currentFadeTime = fadeTime;

	switch (currentFadeMode)
	{
		case AudioFadeType::None: return; // no fade set.
		case AudioFadeType::In: 
			volumeMultiplier = (currentFadeTime / fadeTime) * volume;
			break;
		case AudioFadeType::Out: 
			volumeMultiplier = (1.0f - (currentFadeTime / fadeTime)) * volume;
			break;
	}

	audioComp->SetVolumeMultiplier(volumeMultiplier);
	
	if (currentFadeTime == fadeTime)
	{
		// stop the audio if we are fading out
		if (currentFadeMode == AudioFadeType::Out)
			audioComp->Stop();

		fadeTime = currentAudioPosition = 0;
		currentFadeMode = AudioFadeType::None;
	}
}
```
Now all I had to do was implement this to PlayNextAudioClip and add a new function to stop the audio playing. So, i added a parameter call FadeInLength to playNextAudioClip so it became 

```
void AudioManager::PlayNextAudioClip(float audioPosition, float fadeInLength)
```

then just under start volume I setup the fade

```
{
	//...
	
	float startVolume = volume;
	
	// Setup fade
	if (fadeInLength > 0)
	{
		startVolume = 0;
		fadeTime = fadeInLength;
		currentFadeTime = 0;
		currentFadeMode = AudioFadeType::In;
	}
	
	//...
}
```

Now it was just case of adding StopAudio and pretty much doing the same.

```
void AudioManager::StopAudio(float fadeOutLength)
{
	//Set fade
	if (fadeOutLength > 0)
	{
		fadeTime = fadeOutLength;
		currentFadeMode = AudioFadeType::Out;
	}
	else
	{
		audioComp->Stop();
		audioStarted = false;
	}

}
```

Now we had audio in the game once again except this time it could be looped and faded. It was now time to add bit more basic functionality such as Get and SetVolume, IsLooping, GetAudioLength and AudioFinished.

For AudioFinsihed I added a delegate that I called OnAudioFinished so I could notify the game that its current state needs to update since there is no music playing anymore and it only gets trigger if the audio is not looping (since the audio will never end when looping).

```
void AudioManager::AudioFinished()
{

	if (!IsLooping() && GetAudioPosition() >= GetAudioLength())
		OnAudioFinished.Broadcast();

}
``` 

Finally all that was left to do was set and trigger the music depending on the games state (which I won’t go into too much detail about, since it’s an entire blog post itself), but basically I added an audio clip and fade variables to both the dialogue and main games stats, witch triggers the start audio and the audio is stopped when the end state is triggered, the only difference is that the dialogue state also set its clip to loop since we don’t know how long the player will be on the dialogue and the main game is only as long as the music.

So hopefully this give you an idea how we have applied the audio system to Jazz Odyssey, it was difficult at first since the games time system needed to be in sync with the audio but we got there in the end, and now we have a fairly solid system that gives the player a smooth experience between the dialogue and main game.
