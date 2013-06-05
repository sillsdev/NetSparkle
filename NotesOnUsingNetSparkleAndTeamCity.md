Here are instructions for wiring up .net app, MSBuild script, and TeamCity so that users will be notified when a new version is available. Not only that, but they can read the upcoming release notes before deciding to allow the update. 

#Code
1. Get my [NetSparkle Fork](https://github.com/hatton/NetSparkle), build it, put the dll in your lib, add it to your installer.

2.  Use the instructions in the ReadMe to wire it up to you application. Minimally, this means

<pre>_sparkleApplicationUpdater =
    new Sparkle(@"http://build.palaso.org/guestAuth/repository/download/YourTeamCityConfigurationID/.lastSuccessful/appcast.xml",
		Resources.Bloom);
_sparkleApplicationUpdater.CheckOnFirstApplicationIdle();</pre>

The URL is really the secret sauce. TeamCity lets us get at any exported artifact of a build, withtout having to log in.

Just replace the "YourTeamCityConfigurationID" with what you see in the URL when looking at your TeamCity Configuration. For example, here the id is "bt78":
http://build.palaso.org/admin/editBuild.html?id=buildType:*bt78*


#XML
3.  Create an appcast.xml in your src/Installer folder (this can be anywhere, but this is the convention I'm suggesting for SIL apps). For the contents, copy 
<pre>&lt;?xml version="1.0" encoding="utf-8"?&gt;
&lt;rss version="2.0" xmlns:sparkle="http://www.andymatuschak.org/xml-namespaces/sparkle"  xmlns:dc="http://purl.org/dc/elements/1.1/"&gt;
   &lt;channel&gt;
      &lt;title&gt;Bloom Releases&lt;/title&gt;
      &lt;link&gt;http://bloom.palaso.org/download/&lt;/link&gt;
        &lt;description&gt;Most recent changes with links to updates.&lt;/description&gt;
  		&lt;language&gt;en&lt;/language&gt;
 		&lt;item&gt;
			&lt;title&gt;Version DEV_VERSION_NUMBER&lt;/title&gt;
				&lt;sparkle:releaseNotesLink&gt;
					http://build.palaso.org/guestAuth/repository/download/YourTeamCityConfigurationID/.lastSuccessful/ReleaseNotes.md
				&lt;/sparkle:releaseNotesLink&gt;
			&lt;enclosure url="http://bloom.palaso.org/downloads/BloomInstaller.DEV_VERSION_NUMBER.msi" sparkle:version="DEV_VERSION_NUMBER" type="application/octet-stream" /&gt;
 		&lt;/item&gt;
   &lt;/channel&gt;
&lt;/rss&gt;
</pre>

As before, replace the "YourTeamCityConfigurationID" with what you see in the URL when looking at your TeamCity Configuration. 

#TeamCity
Somewhere in you MSBuild script, have it copy the appcast to your output directory:

<pre>&lt;Copy SourceFiles ="$(RootDir)\src\Installer\appcast.xml"
           DestinationFolder ="$(RootDir)\output\installer"/&gt;</pre>

Now we need to make MSBuild automatically update that file so that the appcast can reflect the new version number, each time you have a successful build.

Presumably you have some way of getting at the version your building. In our PALASO apps, this comes in as a DEV\_VERSION\_NUMBER variable. Somewhere in you MSBuild script, have it use your equivalent in order to update the appcast.xml:

<pre>&lt;FileUpdate File="$(RootDir)\output\installer\appcast.xml"
                 DatePlaceholder='DEV_RELEASE_DATE'
                Regex='DEV_VERSION_NUMBER'
                 ReplacementText ="$(Version)" /&gt;</pre>

Finally, NetSparkle needs to be able to get at the *new* release notes, so that people can see what you've done before deciding to upgrade. You make this possible by publishing your appcast and release notes as artifacts. The URL Configuration ID business above will point to where TeamCity publishes these files.

In you TeamCity Artifacts field, add the readme. I use a markdown format one NetSparkle will take care of converting it to html as needed. Of course you can use text or HTML. For my projects, I end up with

	output\installer\*.msi
	output\installer\appcast.xml
	DistFiles\ReleaseNotes.md
