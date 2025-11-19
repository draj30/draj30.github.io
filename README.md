<apex:page controller="ChatFlowController">
    <style>
        /* Chat Widget Styling */
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

    <button id="launchChatBtn" onclick="document.dispatchEvent(new Event('launchMIAWChatEvent'))">
        Chat with a Specialist
    </button>

    <div id="embedded-messaging-root"></div>

    <script type="text/javascript">
        // Resolve jQuery conflict if you're using jQuery
        var jq = $.noConflict();

        // Initialize the Embedded Messaging service
        function initEmbeddedMessaging() {
            try {
                embeddedservice_bootstrap.settings.language = 'en_US';
                embeddedservice_bootstrap.init(
                    '00D9O000005Id3K',  // Salesforce Org ID
                    'MIDAS_US_CHAT_WEB',  // Chat ID
                    'https://bayeragmiidas--test.sandbox.my.site.com/ESWMIDASUSCHATWEB1736761433673',  // Salesforce URL
                    { scrt2URL: 'https://bayeragmiidas--test.sandbox.my.salesforce-scrt.com' }  // Secure URL
                );
            } catch (err) {
                console.error('Error loading Embedded Messaging: ', err);
            }
        }
    </script>

    <!-- Load Salesforce Embedded Service Script -->
    <script type="text/javascript" 
        src="https://bayeragmiidas--test.sandbox.my.site.com/ESWMIDASUSCHATWEB1736761433673/assets/js/bootstrap.min.js" 
        onload="initEmbeddedMessaging()">
    </script>

    <script>
        window.addEventListener('onEmbeddedMessagingReady', function () {
            embeddedservice_bootstrap.settings.hideChatButtonOnLoad = true;

            var msg = document.getElementById('availabilityMessage');
            var btn = document.getElementById('launchChatBtn');
            var header = document.getElementById('chat-header');
            var card = document.getElementById('chat-widget');

            // Use Salesforce's Embedded Service API to check availability
            function checkAgentAvailability() {
                embeddedservice_bootstrap.utilAPI.checkAgentAvailability()
                    .then(function(result) {
                        if (!msg || !header) return;

                        if (result.status) {
                            if (result.numberofAgent > 0 && result.isWithinBH) {
                                header.style.display = "none";
                                card.style.display = "none"; 
                                msg.innerText = "";
                                msg.classList.remove("offline");
                                btn.style.display = 'block';
                            } else {
                                card.style.display = "block";
                                header.classList.add("offline");

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
                                btn.style.display = 'none';
                            }
                        } else {
                            header.innerText = "⚠️ Error";
                            msg.innerText = 'There was a problem running the availability check.';
                            msg.classList.remove("offline");
                            btn.style.display = 'none';
                        }
                    })
                    .catch(function(err) {
                        console.error('Error checking agent availability:', err);
                        header.innerText = "⚠️ Error";
                        msg.innerText = 'Could not check availability.';
                        msg.classList.remove("offline");
                        btn.style.display = 'none';
                    });
            }

            setInterval(checkAgentAvailability, 60000);
            checkAgentAvailability();

            // Launch chat when custom button is clicked
            document.addEventListener('launchMIAWChatEvent', function () {
                embeddedservice_bootstrap.utilAPI.launchChat()
                    .then(function () { 
                        card.classList.add('hidden');
                    })
                    .catch(function (error) { 
                        msg.innerText = 'Could not launch chat.';
                    });
            });

            // Handle minimize of chat window
            window.addEventListener("onEmbeddedMessagingWindowMinimized", function() {
                var btn = document.getElementById("launchChatBtn");
                if (btn) {
                    btn.style.display = "none"; // Hide custom button when chat is minimized
                }
            });
        });
    </script>
</apex:page>
