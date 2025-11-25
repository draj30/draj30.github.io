<apex:page controller="ChatFlowController" showHeader="false" standardStylesheets="false" sidebar="false">
  <style>
    #chat-widget {
      position: fixed;
      bottom: 20px;
      right: 20px;
      width: 360px;
      background: #fff;
      border-radius: 16px;
      box-shadow: 0 8px 20px rgba(0, 0, 0, 0.15);
      font-family: Arial, sans-serif;
      z-index: 9999;
      overflow: hidden;
      transition: all 0.3s ease-in-out;
    }

    #chat-widget.hidden {
      display: none;
    }

    .chat-header {
      background: rgb(137, 211, 41);
      color: white;
      padding: 14px 18px;
      font-size: 16px;
      font-weight: bold;
      text-align: center;
    }

    .chat-header.offline {
      display: none;
    }

    #availabilityMessage {
      padding: 14px;
      font-size: 14px;
      line-height: 1.6;
      text-align: left;
      background: #fff;
      border-radius: 0 0 16px 16px;
      color: #222;
    }

    .offline-label {
      display: inline-block;
      background: rgb(137, 211, 41);
      color: white;
      padding: 10px 16px;
      font-weight: bold;
      border-radius: 12px;
      margin-bottom: 12px;
      text-align: center;
      width: 100%;
    }

    .chat-hours {
      font-weight: bold;
      color: #000;
      display: block;
      margin: 6px 0 10px 0;
    }

    .submit-link-btn {
      display: inline-block;
      background: rgb(137, 211, 41);
      color: white !important;
      padding: 10px 18px;
      font-size: 14px;
      font-weight: bold;
      border-radius: 10px;
      margin-top: 10px;
      text-decoration: none;
      text-align: center;
    }

    .submit-link-btn:hover {
      background: rgb(110, 175, 33);
    }

    #launchChatBtn {
      background: rgb(137, 211, 41);
      color: white;
      border: none;
      padding: 14px 22px;
      font-size: 16px;
      font-weight: bold;
      border-radius: 12px;
      cursor: pointer;
      display: none;
      width: auto;
      text-align: center;
      transition: all 0.3s ease-in-out;
      position: fixed;
      bottom: 20px;
      right: 20px;
      box-shadow: 0 4px 12px rgba(0,0,0,0.25);
      z-index: 9998;
    }

    #launchChatBtn:hover {
      background: rgb(110, 175, 33);
      transform: translateY(-2px);
      box-shadow: 0 6px 14px rgba(0,0,0,0.3);
    }
  </style>

  <div id="chat-widget">
    <div id="chat-header" class="chat-header">💬 Checking availability...</div>
    <p id="availabilityMessage">Please wait...</p>
  </div>

  <button id="launchChatBtn" onclick="launchChatHandler()">
    Chat with a Specialist
  </button>

  <div id="embedded-messaging-root"></div>

  <script type="text/javascript">
    function initEmbeddedMessaging() {
      try {
        embeddedservice_bootstrap.settings.language = 'en_US';
        embeddedservice_bootstrap.init(
          '00D9O000005Id3K',
          'MIDAS_US_CHAT_WEB',
          'https://bayeragmiidas--test.sandbox.my.site.com/ESWMIDASUSCHATWEB1736761433673',
          { scrt2URL: 'https://bayeragmiidas--test.sandbox.my.salesforce-scrt.com' }
        );
      } catch (err) {
        console.error('Error loading Embedded Messaging: ', err);
      }
    }
  </script>

  <script type="text/javascript" 
    src="https://bayeragmiidas--test.sandbox.my.site.com/ESWMIDASUSCHATWEB1736761433673/assets/js/bootstrap.min.js" 
    onload="initEmbeddedMessaging()">
  </script>

  <script type="text/javascript">
    // Global variables
    var checkIntervalId = null;
    var embeddedMessagingReady = false;

    // REST API Configuration
    var REST_API_CONFIG = {
      endpoint: '/services/apexrest/ChatAvailability',
      sessionId: '{!$Api.Session_ID}'
    };

    /**
     * Call REST API to check agent availability
     */
    function checkAgentAvailabilityREST() {
      return fetch(REST_API_CONFIG.endpoint, {
        method: 'GET',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': 'Bearer ' + REST_API_CONFIG.sessionId
        }
      })
      .then(function(response) {
        if (!response.ok) {
          throw new Error('REST API call failed: ' + response.status);
        }
        return response.json();
      })
      .then(function(result) {
        console.log('API Response:', result);
        return { status: true, data: result };
      })
      .catch(function(error) {
        console.error('Error calling REST API:', error);
        return { status: false, error: error.message };
      });
    }

    /**
     * Update UI based on availability response
     */
    function updateAvailabilityUI(response) {
      var msg = document.getElementById('availabilityMessage');
      var btn = document.getElementById('launchChatBtn');
      var header = document.getElementById('chat-header');
      var card = document.getElementById('chat-widget');

      if (!msg || !header || !card || !btn) {
        console.error('Required DOM elements not found');
        return;
      }

      if (response.status && response.data) {
        var result = response.data;
        
        if (result.numberofAgent && result.numberofAgent > 0 && result.isWithinBH) {
          // Agents available
          header.style.display = "none";
          card.style.display = "none"; 
          msg.innerText = "";
          msg.classList.remove("offline");
          btn.style.display = 'block';
        } else {
          // Agents not available
          card.style.display = "block";
          header.classList.add("offline");
          btn.style.display = 'none';

          if (!result.isWithinBH) {
            msg.innerHTML = "<span class='offline-label'>🕒 Outside Business Hours</span>" +
                            "<div>Hello, our specialists are not available at the moment.<br/>" +
                            "Our live chat and phone hours are:<br/>" +
                            "<span class='chat-hours'>Monday - Friday<br/>8:30AM - 8:00PM EST</span>" +
                            "If you would like, you may submit your question and one of our specialists will follow up with you:<br/>" +
                            "<a class='submit-link-btn' href='https://askmed.bayer.com/submit-question?utm_source=LiveChat&utm_medium=Website&utm_campaign=LiveChat' target='_blank'>Submit Question</a></div>";
          } else {
            msg.innerHTML = "<span class='offline-label'>💬 Agents are Offline</span>" +
                            "<div>I'm sorry, it looks like all agents are unavailable at this time.<br/><br/>" +
                            "Please try again later or submit your question and one of our specialists will follow up with you:<br/>" +
                            "<a class='submit-link-btn' href='https://askmed.bayer.com/submit-question?utm_source=LiveChat&utm_medium=Website&utm_campaign=LiveChat' target='_blank'>Submit Question</a></div>";
          }
          msg.classList.add("offline");
        }
      } else {
        // Error state
        card.style.display = "block";
        header.innerText = "⚠️ Error";
        header.style.display = "block";
        header.classList.remove("offline");
        msg.innerText = 'There was a problem running the availability check.';
        msg.classList.remove("offline");
        btn.style.display = 'none';
      }
    }

    /**
     * Check agent availability
     */
    function checkAgentAvailability() {
      console.log('Checking agent availability...');
      checkAgentAvailabilityREST()
        .then(function(response) {
          updateAvailabilityUI(response);
        })
        .catch(function(error) {
          console.error('Availability check failed:', error);
          updateAvailabilityUI({ status: false, error: error.message });
        });
    }

    /**
     * Launch chat handler
     */
    function launchChatHandler() {
      console.log('Launch chat button clicked');
      checkAgentAvailabilityREST()
        .then(function(response) {
          if (response.status && response.data) {
            var result = response.data;
            
            if (result.numberofAgent && result.numberofAgent > 0) {
              // Launch chat
              if (typeof embeddedservice_bootstrap !== 'undefined' && 
                  embeddedservice_bootstrap.utilAPI && 
                  embeddedservice_bootstrap.utilAPI.launchChat) {
                embeddedservice_bootstrap.utilAPI.launchChat()
                  .then(function() {
                    console.log('Chat launched successfully');
                    var card = document.getElementById('chat-widget');
                    if (card) card.classList.add('hidden');
                  })
                  .catch(function(error) {
                    console.error('Could not launch chat:', error);
                    var msg = document.getElementById('availabilityMessage');
                    if (msg) msg.innerText = 'Could not launch chat. Please try again.';
                  });
              } else {
                console.error('Embedded messaging not ready');
              }
            } else {
              // Show offline message
              updateAvailabilityUI(response);
            }
          } else {
            updateAvailabilityUI({ status: false });
          }
        })
        .catch(function(error) {
          console.error('Launch chat failed:', error);
        });
    }

    /**
     * Initialize when embedded messaging is ready
     */
    window.addEventListener('onEmbeddedMessagingReady', function() {
      console.log('Embedded Messaging is ready');
      embeddedMessagingReady = true;
      
      if (typeof embeddedservice_bootstrap !== 'undefined') {
        embeddedservice_bootstrap.settings.hideChatButtonOnLoad = true;
      }

      // Initial check
      checkAgentAvailability();

      // Check every 60 seconds
      if (checkIntervalId) {
        clearInterval(checkIntervalId);
      }
      checkIntervalId = setInterval(checkAgentAvailability, 60000);
    });

    /**
     * Handle chat minimized event
     */
    window.addEventListener("onEmbeddedMessagingWindowMinimized", function() {
      console.log('Chat window minimized');
      var btn = document.getElementById("launchChatBtn");
      if (btn) {
        btn.style.display = "none";
      }
    });

    /**
     * Fallback: If embedded messaging doesn't load in 10 seconds, run availability check anyway
     */
    setTimeout(function() {
      if (!embeddedMessagingReady) {
        console.log('Running fallback availability check (embedded messaging not ready)');
        checkAgentAvailability();
        
        // Also set up interval
        if (!checkIntervalId) {
          checkIntervalId = setInterval(checkAgentAvailability, 60000);
        }
      }
    }, 10000);
  </script>
</apex:page>
