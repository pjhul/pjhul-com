<svelte:head>
  <title>auth0-web-extension: Easy authentication in Web Extensions with Auth0 | pjhul.com</title>
</svelte:head>

<main class="mx-4 my-6 md:mx-8 space-y-6">
  <div>
    <a href="/" class="text-sm">home</a>
    <span>></span>
    <a href="/blog" class="text-sm">blog</a>
  </div>

  <div>
    <h1 class="text-xl md:text-3xl font-bold">auth0-web-extension: Easy authentication in Web Extensions with Auth0</h1>
    <h4 class="text-sm md:text-base">Friday 28 January 2022</h4>
  </div>

  <section class="text-base md:text-sm space-y-6">
    <p class="text-base font-medium">Check out the library <a href="https://github.com/pjhul/auth0-web-extension">here!</a></p>

    <p class="max-w-2xl">
      Wanted to write this blog post to quickly explain the new library I've been working on for the past few weeks. I created this library out of the
      struggle of trying to get authentication to work within a Chrome extension using MV3. With the recent changes made, most notably changing
      background pages into ephemeral service workers, storing tokens and re-authenticating has become a huge challenge. After running into this over the
      course of my own development, and hoping to avoid having to resort to less desirable methods of storing access tokens like chrome.storage.local<sup><a href="https://developer.chrome.com/docs/extensions/reference/storage/#usage">[1]</a></sup>,
      I decided to write this library. It's primarily aimed at extensions that heavily rely on service workers, particularly those that need to run on every webpage. I'll first discuss how it works, and then go a bit deeper into some of the limitations
      and reasons for the decisions I made.
    </p>

    <p class="max-w-2xl">
      <span class="font-bold">NOTE:</span> <i>This library is still in pre-release and not recommended for production use yet.</i> That being said I would still love for you to play around with it
      and raise any issues you run into over on the <a href="https://github.com/pjhul/auth0-web-extension/issues">issue tracker</a> as I gear up for a production release soon.
    </p>

    <div>
      <h4 class="font-bold">Sections</h4>
      <ol class="ml-6">
        <li><a href="#works">How it Works</a></li>
        <li><a href="#design">Design Considerations</a></li>
        <li><a href="#caveats">Caveats & Limitations</a></li>
        <li><a href="#questions">Questions / Future Development</a></li>
        <li><a href="#end">Would love to hear from you!</a></li>
        <li><a href="#references">References</a></li>
      </ol>
    </div>

    <h3 id="works" class="text-2xl font-bold">How it Works</h3>

    <div class="py-6">
      <pre class="text-xs">
Background Script              Content Script                       Content Script                                  Auth0 SSO
                               (https://wikipedia.org)              (https://joinpatchwork.com)

+---------------+              +---------------------+
|               |              |                     |
| 1. Init Auth  +------------->|  2. Create IFrame   +--------------------------+
|               |              |                     |                          |
+---------------+              +---------------------+                          |  Content scripts loads due to "all_frames" = true
              chrome.tabs.connect                                               |
                                                                                v
+---------------+                                                   +-----------+-------------+
|               |             chrome.runtime.connect                |                         |
| 4. Build URL  |---------------------------------------------------+ 3. Retrive auth params  |
|               |                                                   |                         |
+-------+-------+                                                   +-------------------------+
        |
        |                                                           +-------------------------+               +-------------------+
        |                        port.postMessage                   |                         |               |                   |
        +---------------------------------------------------------->+ 5. Create Auth0 IFrame  +-------------->+   Perform OAuth   |
                                                                    |                         |               |                   |
                                                                    +-------------------------+               +---------+---------+
                                                                                                                        |
+---------------+                                                   +-------------------------+                         |
|               |                port.postMessage                   |                         |                         |
| 7. Get token  +&lt;--------------------------------------------------+ 6. Return code & state  +&lt;------------------------+
|               |                                                   |                         |     window.postMessage
+-------+-------+                                                   +-------------------------+
        |
        |                                  /oauth/token with code and code_verifier
        +---------------------------------------------------------------------------------------------------------------+
                                                                                                                        |
+---------------+                                                                                                       |
|               |                                                                                                       |
|  8. Finish    +&lt;------------------------------------------------------------------------------------------------------+
|               |
+---------------+
      </pre>
      <p class="text-sm italic">i. Flow diagram for the handshake</p>
    </div>

    <p class="max-w-2xl">
      So above you can see my hilariously bad diagram that was <i>hopefully</i> meant to make this easier to understand, so I guess we'll see how well that actually works.
      Anyhow, here is an explanation of each of the 8 steps in the diagram. Later on I'll explain exactly why I've settled on this flow as well as some of the limitations/things
      I'm still figuring out.
    </p>

    <ol class="ml-6 max-w-2xl space-y-4 list-decimal">
      <li>This whole process begins in the background script, since that is eventually where we want to make our authenticated request from. This could begin
        with a message from a content script asking for a specific piece of data or otherwise, but the first step of this process is the background script connects
        to the content script running in the current tab using chrome.runtime.connect. This establishes a long term connection that we will use to pass messages back and
        forth throughout the handshake.
      </li>

      <li>One requirement of this library is that a small big of code needs to be added to each content script which will communicate with the backend and perform
        all the actions where a window and a DOM are needed. When the backend connects with the listener, the next step is for the content script to create an IFrame
        and insert it into the DOM. The 'src' for this IFrame will be the redirect_uri used by the app, which in this example I've set as www.joinpatchwork.com. In this way
        we can perform a normal silent authentication in the context of somewhere the user is already signed in.
      </li>

      <li>After creating this IFrame, the next step is somewhat strange but I think actually quite clever. Instead of trying to communicate with this newly created IFrame, we wait
        for our content script to be injected into it, as one of the requirements for this library is our content script that has the handler in it must have the "all_frames" attribute set to "true"
        in the manifest, meaning it will be immediately added to this new IFrame. Upon being injected into this new IFrame, the first thing our content script does is check if the window origin
        is equal to our redirect url, in which case we know we are running in the newly created iframe. We then connect to our background script using chrome.runtime.connect to retrieve our /authorize url.
      </li>

      <li>Our background script builds the /authorize url using our client_id, redirect_uri, etc. in exactly the same way as the normal auth0-spa-js library. This is then passed back to our content script
        over the same port connection we established last step.
      </li>

      <li>Now that we have all the pieces in place, the process almost exactly mirrors what auth0-spa-js does when getting an access token silently using _getTokenFromIFrame. In fact the code that we run in
        our content script to create the /authorize IFrame is basically identical.
      </li>

      <li>If the user has a current session, we will receive a message from the /authorize IFrame with our code and state, which we then send back to our background script.
      </li>

      <li>We can then finally exchange our code, state, and code_verifier for an access token by calling /oauth/token in the exact same way as in auth0-spa-js.</li>

      <li>We're done! Our access token can then be used however we'd like, and will persist for the life of the service worker.</li>
    </ol>

    <h3 id="design" class="text-2xl font-bold">
      Design Considerations
      <a href="#design">
        #
      </a>
    </h3>

    <h5 class="font-bold">Starting from the background script</h5>
    <p class="max-w-2xl">
      One of my most fundamental design considerations when creating this library was that tokens needed to be kept entirely within the background script. This stems from the fact that the separation between
      content scripts and the page running them isn't <i>super</i> strong, and while normally they do run in separate execution contexts, there is also a good chance that they will be running in the same process as the page.
      Background scripts on the other hand not only run in their own execution contexts, but also run in separate processes<sup><a href="https://developers.google.com/web/ilt/pwa/introduction-to-service-worker">[2]</a></sup>,
      making the barrier between them and the rest of the JS running on webpages much stronger. Now one concern that I still felt lingering was that even though our access token is still only ever available in the background script, there is a lot of messaging back and forth
      with content scripts, and in fact it's a content script that runs the /authorize IFrame and receives the code and state from auth0. We already know that the /authorize IFrame will be created on a webpage which we control (specifically our redirect_uri), but it
      still would be nice to have an extra layer of assurance that no one is able to intercept our code and hijack our token. This problem actually almost solves itself as a result of the oauth flow that Auth0 uses which is called PKCE (read about it <a href="https://auth0.com/docs/get-started/authentication-and-authorization-flow/authorization-code-flow-with-proof-key-for-code-exchange-pkce">here<sup>[3]</sup></a>).
      Basically, even in the very unlikely event that our code is intercepted, an attacker will still be unable to exchange this for an access token without including our code_verifier, which we created in our background script.
      The sha256 hash of this string is included in our /authorize url as our code_challenge, and must be included in our call to /oauth/token in order to get our access token.
    </p>

    <h5 class="font-bold">IFrame within an IFrame</h5>
    <p class="max-w-2xl">
      This is probably the strangest part of the entire flow. We create a hidden IFrame on whatever webpage the user is viewing, and then immediately create a second hidden IFrame <i>within</i> that IFrame. Why can't we just have created the /authorize IFrame directly on the page instead of going through all this trouble?
      Well the answer has to do with redirect URLs. If we were to create our IFrame on whatever page the user is currently browsing (let's say Wikipedia), our request to Auth0 is going to show us trying to use https://wikipedia.org as our redirect_uri, which it will complain
      isn't in our list of allowed redirects. I believe that Auth0 also sets the X-Frame-Ancestors option to only allow embedding of the /authorize url on domains that are set as redirect URLs to prevent clickjacking attacks. We definitely don't want to add redirect URLs that we
      don't control as this would mean those pages could perform login requests on our behalf, and so our only choice is to somehow create this /authorize IFrame from one of our redirect URLs. So immediately my thought was, if Auth0 can create a hidden IFrame for their domain, what if we did the same for ours, 
      and this is where the idea for an IFrame within an IFrame comes from. This also gives the added benefit of allowing us to share the users login state with the extension automatically, e.g. if a user is signed in on https://joinpatchwork.com and we use that as our redirect_uri, this login state will immediately be shared with the extension, allowing users to sign in outside of the extension and have their session shared automatically.
    </p>

    <h5 class="font-bold">No external messaging</h5>
    <p class="max-w-2xl">
      This design consideration was something I wasn't actually sure was possible for quite some time until I remembered the "all_frames" option. Basically, my goal was to avoid the use of onMessageExternal if possible, as this would remove the need to set the "externally_connectable"
      option in the manifest, and also would make it more difficult for an attacker to craft a malicious message as without external messaging they would first need to somehow gain access to the content script execution context. The trickiest step in this process is after we have created the first IFrame
      with our redirect URI, as immediately the question is once we get our code parameter from /authorize, how do we get it back to the background script, and the answer to this is to use the "all_frames" option and have our content script injected into this newly created IFrame and immediately
      create a connection to the backend. This also removes the need for the user of this library to add any extra code to their redirect uri, all they need to do is add an extra line to their content script and make sure it gets injected on their redirect uri.
    </p>

    <h3 id="caveats" class="text-2xl font-bold">
      Caveats & Limitations
      <a href="#caveats">
        #
      </a>
    </h3>

    <h5 class="font-bold">When doesn't this work</h5>
    <p class="max-w-2xl">
      So the biggest limitation of the library right now, is that in order to retrieve an auth token in the background script, there <i>must</i> be at least one content script currently running. While this is a limitation, I've found this still is able to work in most of the typical paradigms for extensions. Any extensions that add in some type of markup to a page and then need to perform authenticated requests when a user interacts with this markup (e.g. Grammarly, Honey, many others) are easily supported.
      These extensions might send a message to the background script asking for some piece of data (e.g. asking to check a paragraph for errors in the Grammarly example), after which the background script would call getTokenSilently, perform this handshake, use the token to query an API, and then return the results back to the content script in a message. This paradigm is by far the most common for web extensions, and now with some of the unfortunate changes in MV3, is
      one of the only paradigms that feels well supported. At the extreme end are extensions that need to run on <i>every</i> webpage, as in the case of the one I'm building, and this is where this library really shines. Since we already need to have a content script on every webpage, this effectively means that we can retrieve an access token in the background at any time.
    </p>

    <p class="max-w-2xl">
      To sum it up, this library won't work well for extensions that do a lot of background work that isn't prompted by a content script and who don't want to request hefty permissions like "Can access all your data on all your sites". However, if you are building something which relies heavily on content scripts making authenticated requests only when a user is on a certain page <i>or</i> if your extension has content scripts on every page, then definitely try this library out.
    </p>

    <div class="py-6">
      <pre class="text-xs">
Content Script                   Background Script                Your API
+------------------+             +------------------+             +-----------------+
|                  +------------>+                  +------------>+                 |
|   User Action    |             | getTokenSilently |             |      Data       |
|                  +&lt;------------+                  +&lt;------------+                 |
+------------------+             +------------------+             +-----------------+
      </pre>
      <p class="text-sm italic">ii. Flow where I think this works best</p>
    </div>

    <h5 class="font-bold">Performance & Persistence</h5>
    <p class="max-w-2xl">
      This is something that in all honesty I expected to be worse. I've done a bit of timing on my laptop, and it seems that on average the first full handshake as shown in the diagram takes just over 1 second. This isn't exactly great, but the nice thing is that on subsequent calls to getTokenSilently, we can pull from the in-memory cache meaning that it consistently takes less than 1ms to get a token. The one caveat here is that since we use an in-memory cache, this only persists for the
      life of the service worker (which unfortunately in MV3 on Chrome looks like is capped at around 5min). This means that if a request is made after the service worker has gone to sleep it will need to perform the whole handshake again with the ~1sec delay. This is something I'm actively looking into improving (see <a href="#questions">questions</a>), but for now it is more than good enough.
    </p>

    <h3 id="questions" class="text-2xl font-bold">
      Questions / Future Development
      <a href="#questions">
        #
      </a>
    </h3>
    <ul class="ml-6 max-w-2xl space-y-4 list-dash">
      <li>Is it safe to store refresh tokens in chrome.storage.local? Right now we only support in-memory caching and don't support refresh tokens, but if we were able to cache refresh tokens in local storage while using RTR that would allow us to consistently re-authenticated across service worker shut-downs.
        This could potentially open a ton of new use cases so I'm actively looking into this but my gut feeling is that chrome.storage.local is not a safe place.
      </li>
      <li>Actively working to improve the reliability of the handshake as well as adding more checks/timeouts to make sure clean-up is reliable as well. This is probably the biggest facet keeping this library from being production ready so I'm focusing a lot on this.
      </li>
    </ul>

    <h3 id="end" class="text-2xl font-bold">
      Would love to hear from you!
      <a href="#end">
        #
      </a>
    </h3>
    <p class="max-w-2xl">
      A lot of the motivation for this project came from my struggles building Patchwork, which is a Chrome Extension that adds instant messaging/sharing to any webpage (check it out if you get the chance). I stumbled across countless deadend forum posts and so my hope is that this library will be able to help people who are in the exact same position I was in. If you have any comments or run into any issues with the library, please raise an issue over at pjhul/auth0-web-extension and I'm
      more than happy to chat/help out!
    </p>

    <h3 id="references" class="text-2xl font-bold">
      References
      <a href="#references">
        #
      </a>
    </h3>

    <ol class="ml-6 space-y-4">
      <li>[1] <a href="https://developer.chrome.com/docs/extensions/reference/storage/#usage">https://developer.chrome.com/docs/extensions/reference/storage/#usage</a></li>
      <li>[2] <a href="https://developers.google.com/web/ilt/pwa/introduction-to-service-worker">https://developers.google.com/web/ilt/pwa/introduction-to-service-worker</a></li>
      <li>[3] <a href="https://auth0.com/docs/get-started/authentication-and-authorization-flow/authorization-code-flow-with-proof-key-for-code-exchange-pkce">https://auth0.com/docs/get-started/authentication-and-authorization-flow/authorization-code-flow-with-proof-key-for-code-exchange-pkce</a></li>
    </ol>
  </section>
</main>




