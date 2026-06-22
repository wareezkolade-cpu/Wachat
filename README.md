<html lang="en"><head>
<!-- Playables SDK -->
<script>// Playables SDK v1.0.0
// Game lifecycle bridge: rAF-based game-ready detection + event communication
(function() {
  'use strict';

  // Idempotency: skip if already initialized (e.g., server-side injection
  // followed by client-side inject-javascript via the Bloks webview component).
  if (window.playablesSDK) return;

  var HANDLER_NAME = 'playablesGameEventHandler';
  var ANDROID_BRIDGE_NAME = '_MetaPlayablesBridge';
  var RAF_FRAME_THRESHOLD = 3;

  var gameReadySent = false;
  var firstInteractionSent = false;
  var errorSent = false;
  var frameCount = 0;
  var originalRAF = window.requestAnimationFrame;

  // --- Transport Layer ---

  function hasIOSBridge() {
    return !!(window.webkit &&
              window.webkit.messageHandlers &&
              window.webkit.messageHandlers[HANDLER_NAME]);
  }

  function hasAndroidBridge() {
    return !!(window[ANDROID_BRIDGE_NAME] &&
              typeof window[ANDROID_BRIDGE_NAME].postEvent === 'function');
  }

  function isInIframe() {
    return !!(window.parent && window.parent !== window);
  }

  function sendEvent(eventName, payload) {
    var message = {
      type: eventName,
      payload: payload || {},
      timestamp: Date.now()
    };

    if (hasIOSBridge()) {
      try {
        window.webkit.messageHandlers[HANDLER_NAME].postMessage(message);
      } catch (e) { /* ignore */ }
      return;
    }

    if (hasAndroidBridge()) {
    try {
      var p = payload || {};
      p.__secureToken = window.__fbAndroidBridgeAuthToken || '';
      p.timestamp = message.timestamp;
      window[ANDROID_BRIDGE_NAME].postEvent(
        eventName,
        JSON.stringify(p)
      );
    } catch (e) { /* ignore */ }
    return;
  }

    if (isInIframe()) {
      try {
        window.parent.postMessage(message, '*');
      } catch (e) { /* ignore */ }
      return;
    }
  }

  // --- rAF Game-Ready Detection ---

  function onFrame() {
    if (gameReadySent) return;

    frameCount++;
    if (frameCount >= RAF_FRAME_THRESHOLD) {
      gameReadySent = true;
      sendEvent('game_ready', {
        frame_count: frameCount,
        detected_at: Date.now()
      });
      return;
    }

    originalRAF.call(window, onFrame);
  }

  if (originalRAF) {
    window.requestAnimationFrame = function(callback) {
      if (!gameReadySent) {
        return originalRAF.call(window, function(timestamp) {
          frameCount++;
          if (frameCount >= RAF_FRAME_THRESHOLD && !gameReadySent) {
            gameReadySent = true;
            sendEvent('game_ready', {
              frame_count: frameCount,
              detected_at: Date.now()
            });
          }
          callback(timestamp);
        });
      }
      return originalRAF.call(window, callback);
    };
  }

  // --- First User Interaction Detection ---

  function setupFirstInteractionDetection() {
    var events = ['touchstart', 'mousedown', 'keydown'];

    function onFirstInteraction() {
      if (firstInteractionSent) return;
      firstInteractionSent = true;
      sendEvent('user_interaction_start', null);

      for (var i = 0; i < events.length; i++) {
        document.removeEventListener(events[i], onFirstInteraction, true);
      }
    }

    for (var i = 0; i < events.length; i++) {
      document.addEventListener(events[i], onFirstInteraction, true);
    }
  }

  if (document.readyState === 'loading') {
    document.addEventListener('DOMContentLoaded', setupFirstInteractionDetection);
  } else {
    setupFirstInteractionDetection();
  }

  // --- Auto Error Capture ---

  window.addEventListener('error', function(event) {
    if (errorSent) return;
    errorSent = true;
    sendEvent('error', {
      message: event.message || 'Unknown error',
      source: event.filename || '',
      lineno: event.lineno || 0,
      colno: event.colno || 0,
      auto_captured: true
    });
  });

  window.addEventListener('unhandledrejection', function(event) {
    if (errorSent) return;
    errorSent = true;
    var reason = event.reason;
    sendEvent('error', {
      message: (reason instanceof Error) ? reason.message : String(reason),
      type: 'unhandled_promise_rejection',
      auto_captured: true
    });
  });

  // --- Public API ---

  window.playablesSDK = {
    complete: function(score) {
      sendEvent('game_ended', {
        score: score,
        completed: true
      });
    },

    error: function(message) {
      if (errorSent) return;
      errorSent = true;
      sendEvent('error', {
        message: message || 'Unknown error',
        auto_captured: false
      });
    },

    sendEvent: function(eventName, payload) {
      if (!eventName || typeof eventName !== 'string') return;
      sendEvent(eventName, payload);
    }
  };

  // Kick off rAF detection in case no game code calls rAF immediately
  if (originalRAF) {
    originalRAF.call(window, onFrame);
  }
})();</script>
<!-- PLAYABLE_TOUCH_PATCH_V1 --><script>
(function() {
  if (window.__playableTouchPatchInstalled) return;
  window.__playableTouchPatchInstalled = true;
  var origAdd = EventTarget.prototype.addEventListener;
  var blockedTypes = { touchstart: 1, touchmove: 1, wheel: 1 };
  EventTarget.prototype.addEventListener = function(type, listener, options) {
    if (blockedTypes[type]) {
      if (options === undefined || options === null) {
        options = { passive: true };
      } else if (typeof options === 'boolean') {
        options = { capture: options, passive: true };
      } else {
        options = Object.assign({}, options, { passive: true });
      }
    }
    return origAdd.call(this, type, listener, options);
  };
})();
</script><script>window.Intl=window.Intl||{};Intl.t=function(s){return(Intl._locale&&Intl._locale[s])||s;};</script>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
<title>Wachat V5 — Realtime Social</title>
<!--
WACHAT V5 - FIREBASE SETUP (5 steps)
1. Go to https://console.firebase.google.com → Create Project (e.g., "wachat-app")
2. Enable: Authentication → Sign-in method → Anonymous (enable)
   Firestore Database → Create database (start in test mode)
   Storage → Get started
3. Project Settings (gear) → Your apps → Web → Register app → Copy config
4. Paste your config below where it says "PASTE_YOURS" (replace entire firebaseConfig)
5. Firestore Rules (for testing - lock down later):
   rules_version='2'; service cloud.firestore { match /databases/{database}/documents { match /{document=**} { allow read, write: if true; } } }
   Storage Rules: allow read, write: if true;
6. Deploy to GitHub Pages / Netlify. For Telegram login, set domain in @BotFather for @Wchart312_bot
-->
<script src="https://cdn.tailwindcss.com"></script>
<link href="https://cdnjs.cloudflare.com/ajax/libs/remixicon/4.2.0/remixicon.min.css" rel="stylesheet">
<style>
  :root { color-scheme: light; }
  body { font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Inter, sans-serif; -webkit-tap-highlight-color: transparent; }
  .no-scrollbar::-webkit-scrollbar{ display:none } .no-scrollbar{ -ms-overflow-style:none; scrollbar-width:none }
  .video-post{ aspect-ratio:9/16; background:#000 }
  .shadow-top{ box-shadow: 0 -1px 3px rgba(0,0,0,.05) }
</style>
<style>*, ::before, ::after{--tw-border-spacing-x:0;--tw-border-spacing-y:0;--tw-translate-x:0;--tw-translate-y:0;--tw-rotate:0;--tw-skew-x:0;--tw-skew-y:0;--tw-scale-x:1;--tw-scale-y:1;--tw-pan-x: ;--tw-pan-y: ;--tw-pinch-zoom: ;--tw-scroll-snap-strictness:proximity;--tw-gradient-from-position: ;--tw-gradient-via-position: ;--tw-gradient-to-position: ;--tw-ordinal: ;--tw-slashed-zero: ;--tw-numeric-figure: ;--tw-numeric-spacing: ;--tw-numeric-fraction: ;--tw-ring-inset: ;--tw-ring-offset-width:0px;--tw-ring-offset-color:#fff;--tw-ring-color:rgb(59 130 246 / 0.5);--tw-ring-offset-shadow:0 0 #0000;--tw-ring-shadow:0 0 #0000;--tw-shadow:0 0 #0000;--tw-shadow-colored:0 0 #0000;--tw-blur: ;--tw-brightness: ;--tw-contrast: ;--tw-grayscale: ;--tw-hue-rotate: ;--tw-invert: ;--tw-saturate: ;--tw-sepia: ;--tw-drop-shadow: ;--tw-backdrop-blur: ;--tw-backdrop-brightness: ;--tw-backdrop-contrast: ;--tw-backdrop-grayscale: ;--tw-backdrop-hue-rotate: ;--tw-backdrop-invert: ;--tw-backdrop-opacity: ;--tw-backdrop-saturate: ;--tw-backdrop-sepia: ;--tw-contain-size: ;--tw-contain-layout: ;--tw-contain-paint: ;--tw-contain-style: }::backdrop{--tw-border-spacing-x:0;--tw-border-spacing-y:0;--tw-translate-x:0;--tw-translate-y:0;--tw-rotate:0;--tw-skew-x:0;--tw-skew-y:0;--tw-scale-x:1;--tw-scale-y:1;--tw-pan-x: ;--tw-pan-y: ;--tw-pinch-zoom: ;--tw-scroll-snap-strictness:proximity;--tw-gradient-from-position: ;--tw-gradient-via-position: ;--tw-gradient-to-position: ;--tw-ordinal: ;--tw-slashed-zero: ;--tw-numeric-figure: ;--tw-numeric-spacing: ;--tw-numeric-fraction: ;--tw-ring-inset: ;--tw-ring-offset-width:0px;--tw-ring-offset-color:#fff;--tw-ring-color:rgb(59 130 246 / 0.5);--tw-ring-offset-shadow:0 0 #0000;--tw-ring-shadow:0 0 #0000;--tw-shadow:0 0 #0000;--tw-shadow-colored:0 0 #0000;--tw-blur: ;--tw-brightness: ;--tw-contrast: ;--tw-grayscale: ;--tw-hue-rotate: ;--tw-invert: ;--tw-saturate: ;--tw-sepia: ;--tw-drop-shadow: ;--tw-backdrop-blur: ;--tw-backdrop-brightness: ;--tw-backdrop-contrast: ;--tw-backdrop-grayscale: ;--tw-backdrop-hue-rotate: ;--tw-backdrop-invert: ;--tw-backdrop-opacity: ;--tw-backdrop-saturate: ;--tw-backdrop-sepia: ;--tw-contain-size: ;--tw-contain-layout: ;--tw-contain-paint: ;--tw-contain-style: }/* ! tailwindcss v3.4.17 | MIT License | https://tailwindcss.com */*,::after,::before{box-sizing:border-box;border-width:0;border-style:solid;border-color:#e5e7eb}::after,::before{--tw-content:''}:host,html{line-height:1.5;-webkit-text-size-adjust:100%;-moz-tab-size:4;tab-size:4;font-family:ui-sans-serif, system-ui, sans-serif, "Apple Color Emoji", "Segoe UI Emoji", "Segoe UI Symbol", "Noto Color Emoji";font-feature-settings:normal;font-variation-settings:normal;-webkit-tap-highlight-color:transparent}body{margin:0;line-height:inherit}hr{height:0;color:inherit;border-top-width:1px}abbr:where([title]){-webkit-text-decoration:underline dotted;text-decoration:underline dotted}h1,h2,h3,h4,h5,h6{font-size:inherit;font-weight:inherit}a{color:inherit;text-decoration:inherit}b,strong{font-weight:bolder}code,kbd,pre,samp{font-family:ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, "Liberation Mono", "Courier New", monospace;font-feature-settings:normal;font-variation-settings:normal;font-size:1em}small{font-size:80%}sub,sup{font-size:75%;line-height:0;position:relative;vertical-align:baseline}sub{bottom:-.25em}sup{top:-.5em}table{text-indent:0;border-color:inherit;border-collapse:collapse}button,input,optgroup,select,textarea{font-family:inherit;font-feature-settings:inherit;font-variation-settings:inherit;font-size:100%;font-weight:inherit;line-height:inherit;letter-spacing:inherit;color:inherit;margin:0;padding:0}button,select{text-transform:none}button,input:where([type=button]),input:where([type=reset]),input:where([type=submit]){-webkit-appearance:button;background-color:transparent;background-image:none}:-moz-focusring{outline:auto}:-moz-ui-invalid{box-shadow:none}progress{vertical-align:baseline}::-webkit-inner-spin-button,::-webkit-outer-spin-button{height:auto}[type=search]{-webkit-appearance:textfield;outline-offset:-2px}::-webkit-search-decoration{-webkit-appearance:none}::-webkit-file-upload-button{-webkit-appearance:button;font:inherit}summary{display:list-item}blockquote,dd,dl,figure,h1,h2,h3,h4,h5,h6,hr,p,pre{margin:0}fieldset{margin:0;padding:0}legend{padding:0}menu,ol,ul{list-style:none;margin:0;padding:0}dialog{padding:0}textarea{resize:vertical}input::placeholder,textarea::placeholder{opacity:1;color:#9ca3af}[role=button],button{cursor:pointer}:disabled{cursor:default}audio,canvas,embed,iframe,img,object,svg,video{display:block;vertical-align:middle}img,video{max-width:100%;height:auto}[hidden]:where(:not([hidden=until-found])){display:none}.fixed{position:fixed}.absolute{position:absolute}.relative{position:relative}.sticky{position:sticky}.left-0{left:0px}.right-0{right:0px}.top-0{top:0px}.bottom-0{bottom:0px}.right-2{right:0.5rem}.right-7{right:1.75rem}.top-2{top:0.5rem}.z-40{z-index:40}.z-10{z-index:10}.z-30{z-index:30}.mx-auto{margin-left:auto;margin-right:auto}.mb-2{margin-bottom:0.5rem}.ml-auto{margin-left:auto}.mt-3{margin-top:0.75rem}.-ml-1{margin-left:-0.25rem}.-mt-0\.5{margin-top:-0.125rem}.mr-1{margin-right:0.25rem}.mr-3{margin-right:0.75rem}.mt-1{margin-top:0.25rem}.mt-4{margin-top:1rem}.block{display:block}.flex{display:flex}.inline-flex{display:inline-flex}.grid{display:grid}.hidden{display:none}.h-10{height:2.5rem}.h-14{height:3.5rem}.h-2{height:0.5rem}.h-1\.5{height:0.375rem}.h-24{height:6rem}.h-8{height:2rem}.h-9{height:2.25rem}.h-\[68px\]{height:68px}.h-\[calc\(100vh-112px\)\]{height:calc(100vh - 112px)}.h-full{height:100%}.max-h-\[360px\]{max-height:360px}.min-h-screen{min-height:100vh}.min-h-0{min-height:0px}.w-10{width:2.5rem}.w-2{width:0.5rem}.w-full{width:100%}.w-24{width:6rem}.w-8{width:2rem}.w-9{width:2.25rem}.w-14{width:3.5rem}.min-w-0{min-width:0px}.min-w-\[64px\]{min-width:64px}.max-w-\[500px\]{max-width:500px}.flex-1{flex:1 1 0%}.shrink-0{flex-shrink:0}@keyframes pulse{50%{opacity:.5}}.animate-pulse{animation:pulse 2s cubic-bezier(0.4, 0, 0.6, 1) infinite}.cursor-pointer{cursor:pointer}.resize-none{resize:none}.grid-cols-4{grid-template-columns:repeat(4, minmax(0, 1fr))}.flex-col{flex-direction:column}.place-items-center{place-items:center}.items-center{align-items:center}.justify-center{justify-content:center}.justify-between{justify-content:space-between}.gap-2{gap:0.5rem}.gap-3{gap:0.75rem}.gap-1{gap:0.25rem}.gap-5{gap:1.25rem}.gap-1\.5{gap:0.375rem}.gap-2\.5{gap:0.625rem}.gap-6{gap:1.5rem}.space-y-3 > :not([hidden]) ~ :not([hidden]){--tw-space-y-reverse:0;margin-top:calc(0.75rem * calc(1 - var(--tw-space-y-reverse)));margin-bottom:calc(0.75rem * var(--tw-space-y-reverse))}.space-y-2 > :not([hidden]) ~ :not([hidden]){--tw-space-y-reverse:0;margin-top:calc(0.5rem * calc(1 - var(--tw-space-y-reverse)));margin-bottom:calc(0.5rem * var(--tw-space-y-reverse))}.divide-y > :not([hidden]) ~ :not([hidden]){--tw-divide-y-reverse:0;border-top-width:calc(1px * calc(1 - var(--tw-divide-y-reverse)));border-bottom-width:calc(1px * var(--tw-divide-y-reverse))}.overflow-hidden{overflow:hidden}.overflow-x-auto{overflow-x:auto}.overflow-y-auto{overflow-y:auto}.truncate{overflow:hidden;text-overflow:ellipsis;white-space:nowrap}.whitespace-pre-wrap{white-space:pre-wrap}.break-all{word-break:break-all}.rounded-2xl{border-radius:1rem}.rounded-full{border-radius:9999px}.rounded-xl{border-radius:0.75rem}.rounded-lg{border-radius:0.5rem}.border-2{border-width:2px}.border-b{border-bottom-width:1px}.border-t{border-top-width:1px}.border-dashed{border-style:dashed}.border-\[\#1877f2\]{--tw-border-opacity:1;border-color:rgb(24 119 242 / var(--tw-border-opacity, 1))}.bg-\[\#f0f2f5\]{--tw-bg-opacity:1;background-color:rgb(240 242 245 / var(--tw-bg-opacity, 1))}.bg-amber-400{--tw-bg-opacity:1;background-color:rgb(251 191 36 / var(--tw-bg-opacity, 1))}.bg-black{--tw-bg-opacity:1;background-color:rgb(0 0 0 / var(--tw-bg-opacity, 1))}.bg-gray-200{--tw-bg-opacity:1;background-color:rgb(229 231 235 / var(--tw-bg-opacity, 1))}.bg-white{--tw-bg-opacity:1;background-color:rgb(255 255 255 / var(--tw-bg-opacity, 1))}.bg-yellow-300{--tw-bg-opacity:1;background-color:rgb(253 224 71 / var(--tw-bg-opacity, 1))}.bg-\[\#1877f2\]{--tw-bg-opacity:1;background-color:rgb(24 119 242 / var(--tw-bg-opacity, 1))}.bg-black\/70{background-color:rgb(0 0 0 / 0.7)}.bg-gray-50{--tw-bg-opacity:1;background-color:rgb(249 250 251 / var(--tw-bg-opacity, 1))}.bg-green-50{--tw-bg-opacity:1;background-color:rgb(240 253 244 / var(--tw-bg-opacity, 1))}.bg-green-500{--tw-bg-opacity:1;background-color:rgb(34 197 94 / var(--tw-bg-opacity, 1))}.bg-red-500{--tw-bg-opacity:1;background-color:rgb(239 68 68 / var(--tw-bg-opacity, 1))}.bg-white\/20{background-color:rgb(255 255 255 / 0.2)}.bg-\[\#e7f0ff\]{--tw-bg-opacity:1;background-color:rgb(231 240 255 / var(--tw-bg-opacity, 1))}.bg-green-400{--tw-bg-opacity:1;background-color:rgb(74 222 128 / var(--tw-bg-opacity, 1))}.object-cover{object-fit:cover}.p-3{padding:0.75rem}.p-4{padding:1rem}.p-1{padding:0.25rem}.p-2{padding:0.5rem}.p-2\.5{padding:0.625rem}.p-6{padding:1.5rem}.p-8{padding:2rem}.px-2{padding-left:0.5rem;padding-right:0.5rem}.px-3{padding-left:0.75rem;padding-right:0.75rem}.px-4{padding-left:1rem;padding-right:1rem}.py-1\.5{padding-top:0.375rem;padding-bottom:0.375rem}.py-24{padding-top:6rem;padding-bottom:6rem}.py-3{padding-top:0.75rem;padding-bottom:0.75rem}.px-6{padding-left:1.5rem;padding-right:1.5rem}.py-1{padding-top:0.25rem;padding-bottom:0.25rem}.py-2{padding-top:0.5rem;padding-bottom:0.5rem}.py-2\.5{padding-top:0.625rem;padding-bottom:0.625rem}.py-3\.5{padding-top:0.875rem;padding-bottom:0.875rem}.pb-20{padding-bottom:5rem}.pt-14{padding-top:3.5rem}.pb-3{padding-bottom:0.75rem}.pt-3{padding-top:0.75rem}.pb-2{padding-bottom:0.5rem}.text-left{text-align:left}.text-center{text-align:center}.text-4xl{font-size:2.25rem;line-height:2.5rem}.text-\[12px\]{font-size:12px}.text-\[15px\]{font-size:15px}.text-\[16px\]{font-size:16px}.text-\[26px\]{font-size:26px}.text-sm{font-size:0.875rem;line-height:1.25rem}.text-2xl{font-size:1.5rem;line-height:2rem}.text-\[11px\]{font-size:11px}.text-\[13px\]{font-size:13px}.text-\[18px\]{font-size:18px}.text-\[24px\]{font-size:24px}.text-lg{font-size:1.125rem;line-height:1.75rem}.text-xl{font-size:1.25rem;line-height:1.75rem}.text-xs{font-size:0.75rem;line-height:1rem}.text-\[14px\]{font-size:14px}.font-extrabold{font-weight:800}.font-medium{font-weight:500}.font-bold{font-weight:700}.font-semibold{font-weight:600}.uppercase{text-transform:uppercase}.leading-snug{line-height:1.375}.leading-tight{line-height:1.25}.tracking-tight{letter-spacing:-0.025em}.tracking-wide{letter-spacing:0.025em}.text-black{--tw-text-opacity:1;color:rgb(0 0 0 / var(--tw-text-opacity, 1))}.text-gray-500{--tw-text-opacity:1;color:rgb(107 114 128 / var(--tw-text-opacity, 1))}.text-gray-900{--tw-text-opacity:1;color:rgb(17 24 39 / var(--tw-text-opacity, 1))}.text-white{--tw-text-opacity:1;color:rgb(255 255 255 / var(--tw-text-opacity, 1))}.text-\[\#1877f2\]{--tw-text-opacity:1;color:rgb(24 119 242 / var(--tw-text-opacity, 1))}.text-\[\#229ED9\]{--tw-text-opacity:1;color:rgb(34 158 217 / var(--tw-text-opacity, 1))}.text-gray-400{--tw-text-opacity:1;color:rgb(156 163 175 / var(--tw-text-opacity, 1))}.text-gray-600{--tw-text-opacity:1;color:rgb(75 85 99 / var(--tw-text-opacity, 1))}.text-green-700{--tw-text-opacity:1;color:rgb(21 128 61 / var(--tw-text-opacity, 1))}.text-red-600{--tw-text-opacity:1;color:rgb(220 38 38 / var(--tw-text-opacity, 1))}.text-gray-700{--tw-text-opacity:1;color:rgb(55 65 81 / var(--tw-text-opacity, 1))}.placeholder-gray-500::placeholder{--tw-placeholder-opacity:1;color:rgb(107 114 128 / var(--tw-placeholder-opacity, 1))}.opacity-90{opacity:0.9}.shadow-sm{--tw-shadow:0 1px 2px 0 rgb(0 0 0 / 0.05);--tw-shadow-colored:0 1px 2px 0 var(--tw-shadow-color);box-shadow:var(--tw-ring-offset-shadow, 0 0 #0000), var(--tw-ring-shadow, 0 0 #0000), var(--tw-shadow)}.outline-none{outline:2px solid transparent;outline-offset:2px}.ring-2{--tw-ring-offset-shadow:var(--tw-ring-inset) 0 0 0 var(--tw-ring-offset-width) var(--tw-ring-offset-color);--tw-ring-shadow:var(--tw-ring-inset) 0 0 0 calc(2px + var(--tw-ring-offset-width)) var(--tw-ring-color);box-shadow:var(--tw-ring-offset-shadow), var(--tw-ring-shadow), var(--tw-shadow, 0 0 #0000)}.ring-\[\#1877f2\]\/20{--tw-ring-color:rgb(24 119 242 / 0.2)}.transition-all{transition-property:all;transition-timing-function:cubic-bezier(0.4, 0, 0.2, 1);transition-duration:150ms}.hover\:bg-gray-50:hover{--tw-bg-opacity:1;background-color:rgb(249 250 251 / var(--tw-bg-opacity, 1))}.hover\:text-\[\#1877f2\]:hover{--tw-text-opacity:1;color:rgb(24 119 242 / var(--tw-text-opacity, 1))}.disabled\:opacity-50:disabled{opacity:0.5}</style></head>
<body class="bg-[#f0f2f5] text-[15px] text-gray-900">
<div class="max-w-[500px] mx-auto min-h-screen bg-[#f0f2f5] relative pb-20">

  <!-- Header -->
  <header class="fixed top-0 left-0 right-0 z-40" style="background:#1877f2">
    <div class="max-w-[500px] mx-auto h-14 flex items-center px-4 text-white">
      <h1 class="text-[26px] font-extrabold tracking-tight">wachat</h1>
      <div class="ml-auto flex items-center gap-2">
        <span id="statusDot" class="w-2 h-2 bg-green-400 rounded-full"></span>
        <span id="headerName" class="text-sm font-medium">Guest User</span>
      </div>
    <div id="demoBanner" class="bg-amber-400 text-black text-center text-[12px] py-1.5 px-2">Demo Mode — Add your Firebase config to enable live features</div>
  </div></header>

  <main class="pt-14">
    <!-- FEED -->
    <section id="tab-feed" class="">
      <div class="bg-white border-b">
        <div id="stories" class="flex gap-3 overflow-x-auto no-scrollbar px-3 py-3">
    <button class="flex flex-col items-center gap-1 min-w-[64px]" id="addStoryBtn">
      <div class="w-14 h-14 rounded-full bg-[#e7f0ff] grid place-items-center border-2 border-[#1877f2] border-dashed"><i class="ri-add-line text-[#1877f2] text-2xl"></i></div>
      <span class="text-[11px] text-gray-700">Your story</span>
    </button>
    
  </div>
      </div>
      <div id="postsContainer" class="space-y-3 mt-3 px-2">
    <article class="bg-white rounded-2xl shadow-sm overflow-hidden">
      <div class="p-3 flex items-center gap-2.5">
        <img src="undefined" class="w-9 h-9 rounded-full object-cover">
        <div class="flex-1 min-w-0">
          <div class="font-semibold text-[14px] leading-tight">Wachat Team</div>
          <div class="text-[12px] text-gray-500">@ · 2d</div>
        </div>
        <i class="ri-more-line text-gray-400"></i>
      </div>
      <p class="px-3 pb-2 text-[15px] leading-snug whitespace-pre-wrap">Welcome to Wachat! 🇳🇬 No data no wahala — everything works offline for inside your phone. No internet, no problem!</p>
      
      <div class="px-3 py-2.5 flex items-center gap-6 text-gray-600 border-t">
        <button class="flex items-center gap-1.5 hover:text-[#1877f2]" onclick="window.likePost('1781926427303')"><i class="ri-heart-3-line text-xl"></i><span class="text-sm">24</span></button>
        <button class="flex items-center gap-1.5"><i class="ri-chat-1-line text-xl"></i><span class="text-sm">Reply</span></button>
        <button class="flex items-center gap-1.5 ml-auto"><i class="ri-share-forward-line text-xl"></i></button>
      </div>
    </article>
  
    <article class="bg-white rounded-2xl shadow-sm overflow-hidden">
      <div class="p-3 flex items-center gap-2.5">
        <img src="undefined" class="w-9 h-9 rounded-full object-cover">
        <div class="flex-1 min-w-0">
          <div class="font-semibold text-[14px] leading-tight">Amaka</div>
          <div class="text-[12px] text-gray-500">@ · 1d</div>
        </div>
        <i class="ri-more-line text-gray-400"></i>
      </div>
      <p class="px-3 pb-2 text-[15px] leading-snug whitespace-pre-wrap">NEPA just take light but my gist still dey go 😂 Who dey online?</p>
      
      <div class="px-3 py-2.5 flex items-center gap-6 text-gray-600 border-t">
        <button class="flex items-center gap-1.5 hover:text-[#1877f2]" onclick="window.likePost('1782012827303')"><i class="ri-heart-3-line text-xl"></i><span class="text-sm">12</span></button>
        <button class="flex items-center gap-1.5"><i class="ri-chat-1-line text-xl"></i><span class="text-sm">Reply</span></button>
        <button class="flex items-center gap-1.5 ml-auto"><i class="ri-share-forward-line text-xl"></i></button>
      </div>
    </article>
  
    <article class="bg-white rounded-2xl shadow-sm overflow-hidden">
      <div class="p-3 flex items-center gap-2.5">
        <img src="undefined" class="w-9 h-9 rounded-full object-cover">
        <div class="flex-1 min-w-0">
          <div class="font-semibold text-[14px] leading-tight">Tunde</div>
          <div class="text-[12px] text-gray-500">@ · 17h</div>
        </div>
        <i class="ri-more-line text-gray-400"></i>
      </div>
      <p class="px-3 pb-2 text-[15px] leading-snug whitespace-pre-wrap">First time using Wachat. E sharp die! No need data.</p>
      
      <div class="px-3 py-2.5 flex items-center gap-6 text-gray-600 border-t">
        <button class="flex items-center gap-1.5 hover:text-[#1877f2]" onclick="window.likePost('1782081227303')"><i class="ri-heart-3-line text-xl"></i><span class="text-sm">7</span></button>
        <button class="flex items-center gap-1.5"><i class="ri-chat-1-line text-xl"></i><span class="text-sm">Reply</span></button>
        <button class="flex items-center gap-1.5 ml-auto"><i class="ri-share-forward-line text-xl"></i></button>
      </div>
    </article>
  </div>
      <div id="emptyPosts" class="hidden text-center py-24 text-gray-500">
        <i class="ri-compass-3-line text-4xl mb-2 block"></i>
        No posts yet<br><span class="text-sm">Be the first to post!</span>
      </div>
    </section>

    <!-- POST -->
    <section id="tab-post" class="hidden p-3">
      <div class="bg-white rounded-2xl shadow-sm p-4">
        <div class="flex gap-3">
          <img id="postAvatar" class="w-10 h-10 rounded-full bg-gray-200 object-cover" alt="" src="https://ui-avatars.com/api/?name=Guest&amp;background=1877f2&amp;color=fff">
          <textarea id="postText" class="flex-1 resize-none outline-none text-[16px] placeholder-gray-500 leading-snug" rows="4" placeholder="Wetin dey sup?"></textarea>
        </div>
        <div id="videoPreview" class="hidden mt-3 relative">
          <video id="previewVideo" class="w-full rounded-xl max-h-[360px] bg-black" playsinline="" controls=""></video>
          <button id="removeVideo" class="absolute top-2 right-2 bg-black/70 text-white rounded-full w-8 h-8 grid place-items-center"><i class="ri-close-line"></i></button>
        </div>
        <div id="uploadProgress" class="hidden mt-3">
          <div class="w-full bg-gray-200 rounded-full h-1.5"><div id="uploadBar" class="h-1.5 rounded-full transition-all" style="width:0%;background:#1877f2"></div></div>
          <p id="uploadText" class="text-xs text-gray-500 mt-1">Uploading video...</p>
        </div>
        <div class="flex items-center justify-between mt-4 pt-3 border-t">
          <div class="flex gap-5">
            <button class="text-gray-600 flex items-center gap-1" onclick="alert('Photo posts coming in V5.1')"><i class="ri-image-add-line text-xl"></i><span class="text-sm">Photo</span></button>
            <label class="text-gray-600 flex items-center gap-1 cursor-pointer">
              <i class="ri-vidicon-line text-xl"></i><span class="text-sm">Video</span>
              <input id="videoInput" type="file" accept="video/*" class="hidden">
            </label>
          </div>
          <button id="postBtn" class="px-6 py-1.5 rounded-full text-white font-semibold disabled:opacity-50" style="background:#1877f2">Post</button>
        </div>
      </div>
      <p class="text-[12px] text-gray-500 text-center mt-3">Posts are public. Be kind.</p>
    </section>

    <!-- CHAT -->
    <section id="tab-chat" class="hidden h-[calc(100vh-112px)] flex flex-col">
      <div id="chatListView" class="flex-1 flex flex-col min-h-0">
        <div class="bg-white px-4 py-3 border-b flex items-center justify-between sticky top-0 z-10">
          <h2 class="font-bold text-[18px]">Chats</h2>
          <button id="addFriendBtn" class="text-[#1877f2] text-sm font-semibold"><i class="ri-user-add-line mr-1"></i>Add Friend</button>
        </div>
        <div class="bg-white border-b">
          <div class="px-4 py-2 text-[12px] text-gray-500 uppercase tracking-wide">Friends</div>
          <div class="px-3 pb-3">
            <div id="friendsList" class="flex gap-3 overflow-x-auto no-scrollbar"><div class="text-[12px] text-gray-400 py-2">Tap "Add Friend" to start</div></div>
          </div>
        </div>
        <div id="chatsContainer" class="flex-1 overflow-y-auto bg-white divide-y"><div class="p-8 text-center text-gray-500 text-sm">No chats yet<br> Add a friend to start messaging</div></div>
      </div>

      <div id="chatView" class="hidden flex-1 flex flex-col h-full bg-[#f0f2f5] min-h-0">
        <div class="bg-[#1877f2] text-white px-3 h-14 flex items-center gap-3 shrink-0">
          <button id="chatBack" class="p-1 -ml-1"><i class="ri-arrow-left-line text-2xl"></i></button>
          <img id="chatAvatar" class="w-9 h-9 rounded-full bg-white/20 object-cover">
          <div class="flex-1 min-w-0">
            <div id="chatName" class="font-semibold leading-tight truncate"></div>
            <div id="chatStatus" class="text-[11px] opacity-90"></div>
          </div>
        </div>
        <div id="messagesContainer" class="flex-1 overflow-y-auto p-3 space-y-2"></div>
        <div class="bg-white p-2.5 border-t flex items-center gap-2 shrink-0">
          <input id="messageInput" class="flex-1 bg-[#f0f2f5] rounded-full px-4 py-2.5 outline-none text-[15px]" placeholder="Message">
          <button id="sendBtn" class="w-10 h-10 rounded-full grid place-items-center text-white shrink-0" style="background:#1877f2"><i class="ri-send-plane-2-fill text-lg"></i></button>
        </div>
      </div>
    </section>

    <!-- ME -->
    <section id="tab-me" class="hidden p-3">
      <div class="bg-white rounded-2xl shadow-sm p-6 text-center">
        <img id="meAvatar" class="w-24 h-24 rounded-full mx-auto object-cover ring-2 ring-[#1877f2]/20" src="https://ui-avatars.com/api/?name=Guest&amp;background=1877f2&amp;color=fff">
        <h3 id="meName" class="mt-3 font-bold text-xl">Guest User</h3>
        <p id="meUsername" class="text-gray-500 -mt-0.5">@guest639</p>
        <div class="mt-3 inline-flex items-center gap-2 px-3 py-1 bg-green-50 rounded-full">
          <span class="w-2 h-2 bg-green-500 rounded-full animate-pulse"></span>
          <span class="text-xs text-green-700 font-medium" id="meStatus">Online</span>
        </div>
      </div>

      <div class="bg-white rounded-2xl shadow-sm mt-3 p-4">
        <label class="text-[11px] text-gray-500 uppercase tracking-wide">Firebase UID</label>
        <div class="flex items-center gap-2 mt-1">
          <code id="meUid" class="text-[12px] flex-1 bg-gray-50 p-2.5 rounded-lg break-all">guest_l4h57q</code>
          <button onclick="navigator.clipboard.writeText(document.getElementById('meUid').textContent); this.innerHTML='<i class=\'ri-check-line text-green-600\'></i>'" class="text-gray-500 p-2"><i class="ri-file-copy-line"></i></button>
        </div>
      </div>

      <div class="bg-white rounded-2xl shadow-sm mt-3 divide-y">
        <button id="editProfileBtn" class="w-full text-left px-4 py-3.5 hover:bg-gray-50 flex items-center"><i class="ri-edit-box-line mr-3 text-xl text-gray-600"></i><span>Edit Profile</span></button>
        <div class="px-4 py-3">
          <div class="text-[13px] text-gray-600 mb-2 flex items-center gap-2"><i class="ri-telegram-fill text-[#229ED9]"></i> Connect Telegram (optional)</div>
          <div id="telegramContainer"></div>
          <p class="text-[11px] text-gray-400 mt-1">Bot: @Wchart312_bot — requires domain whitelist in BotFather</p>
        </div>
        <button id="logoutBtn" class="w-full text-left px-4 py-3.5 hover:bg-gray-50 flex items-center text-red-600"><i class="ri-logout-box-r-line mr-3 text-xl"></i><span>Logout</span></button>
      </div>
    </section>
  </main>

  <!-- Bottom Nav -->
  <nav class="fixed bottom-0 left-0 right-0 z-30 bg-white shadow-top border-t">
    <div class="max-w-[500px] mx-auto grid grid-cols-4 h-[68px] pb-safe">
      <button data-tab="feed" class="nav-btn flex flex-col items-center justify-center gap-1 text-[#1877f2]"><i class="ri-home-5-fill text-[24px]"></i><span class="text-[11px] font-medium">Feed</span></button>
      <button data-tab="post" class="nav-btn flex flex-col items-center justify-center gap-1 text-gray-500"><i class="ri-add-box-line text-[24px]"></i><span class="text-[11px]">Post</span></button>
      <button data-tab="chat" class="nav-btn flex flex-col items-center justify-center gap-1 text-gray-500 relative"><i class="ri-chat-3-line text-[24px]"></i><span class="text-[11px]">Chat</span><span id="chatBadge" class="hidden absolute top-2 right-7 w-2 h-2 bg-red-500 rounded-full"></span></button>
      <button data-tab="me" class="nav-btn flex flex-col items-center justify-center gap-1 text-gray-500"><i class="ri-user-3-line text-[24px]"></i><span class="text-[11px]">Me</span></button>
    </div>
  </nav>
</div>

<script type="module">
/* ========= FIREBASE CONFIG - REPLACE WITH YOURS ========= */
const firebaseConfig = {
  apiKey: "PASTE_YOURS",
  authDomain: "wachat-app.firebaseapp.com",
  projectId: "wachat-app",
  storageBucket: "wachat-app.appspot.com",
  messagingSenderId: "...",
  appId: "..."
};
/* ======================================================== */

import { initializeApp } from 'https://www.gstatic.com/firebasejs/10.12.2/firebase-app.js';
import { getFirestore, collection, addDoc, onSnapshot, query, orderBy, serverTimestamp, doc, updateDoc, increment, where, getDoc, setDoc, getDocs, limit } from 'https://www.gstatic.com/firebasejs/10.12.2/firebase-firestore.js';
import { getAuth, signInAnonymously, onAuthStateChanged, signOut } from 'https://www.gstatic.com/firebasejs/10.12.2/firebase-auth.js';
import { getStorage, ref, uploadBytesResumable, getDownloadURL } from 'https://www.gstatic.com/firebasejs/10.12.2/firebase-storage.js';

const DEMO_MODE = firebaseConfig.apiKey === "PASTE_YOURS";
let app, db, auth, storage;
if(!DEMO_MODE){
  app = initializeApp(firebaseConfig);
  db = getFirestore(app);
  auth = getAuth(app);
  storage = getStorage(app);
}

const $ = id => document.getElementById(id);
const esc = s => (s||'').replace(/[&<>"']/g, c => ({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'}[c]));
const timeAgo = ts => { const d = ts?.toMillis ? ts.toMillis() : ts; if(!d) return 'now'; const s = Math.floor((Date.now()-d)/1000); if(s<60) return 'now'; if(s<3600) return Math.floor(s/60)+'m'; if(s<86400) return Math.floor(s/3600)+'h'; return Math.floor(s/86400)+'d'; };

const state = {
  user: null,
  postsUnsub: null,
  chatsUnsub: null,
  msgsUnsub: null,
  friendsUnsub: null,
  currentChatId: null,
  currentFriend: null,
  pendingVideoFile: null,
};

function initDemoData(){
  if(!localStorage.getItem('wachat_posts')){
    localStorage.setItem('wachat_posts', JSON.stringify([
      {id:'p1', userId:'d1', name:'Aisha Bello', username:'aisha', avatar:'https://ui-avatars.com/api/?name=Aisha+Bello&background=1877f2&color=fff', text:'Wetin dey sup guys! Welcome to Wachat V5 🎉', videoUrl:'', timestamp:Date.now()-3600000, likes:24},
      {id:'p2', userId:'d2', name:'Tunde Okafor', username:'tunde', avatar:'https://ui-avatars.com/api/?name=Tunde+Okafor&background=10b981&color=fff', text:'My first video on Wachat!', videoUrl:'https://storage.googleapis.com/gtv-videos-bucket/sample/BigBuckBunny.mp4', timestamp:Date.now()-1800000, likes:12},
    ]));
  }
  if(!localStorage.getItem('wachat_chats')){
    localStorage.setItem('wachat_chats', JSON.stringify([]));
  }
}

async function initApp(){
  $('demoBanner').classList.toggle('hidden', !DEMO_MODE);
  
  if(DEMO_MODE){
    state.user = JSON.parse(localStorage.getItem('wachat_user')) || {
      uid: 'guest_' + Math.random().toString(36).slice(2,8),
      name: 'Guest User',
      username: 'guest'+Math.floor(Math.random()*900+100),
      avatar: 'https://ui-avatars.com/api/?name=Guest&background=1877f2&color=fff',
      online: true
    };
    localStorage.setItem('wachat_user', JSON.stringify(state.user));
    updateMeUI();
    $('headerName').textContent = state.user.name;
    $('statusDot').className = 'w-2 h-2 bg-green-400 rounded-full';
    initDemoData();
    listenPosts(); listenChats(); listenFriends(); renderStories(); setupUI();
    return;
  }

  onAuthStateChanged(auth, async (fbUser) => {
    if(fbUser){
      state.user = await getOrCreateUser(fbUser);
      updateMeUI();
      $('headerName').textContent = state.user.name;
      $('statusDot').className = 'w-2 h-2 bg-green-400 rounded-full';
      listenPosts(); listenChats(); listenFriends(); renderStories(); setupUI(); setupTelegram();
      // presence
      window.addEventListener('beforeunload', () => updateDoc(doc(db,'users',state.user.uid), {online:false, lastSeen:serverTimestamp()}).catch(()=>{}));
    } else {
      await signInAnonymously(auth).catch(e=>{ console.error(e); alert('Enable Anonymous Auth in Firebase') });
    }
  });
}

async function getOrCreateUser(fbUser){
  const uref = doc(db,'users',fbUser.uid);
  const snap = await getDoc(uref);
  if(snap.exists()){
    await updateDoc(uref, {online:true, lastSeen:serverTimestamp()});
    return {uid:fbUser.uid, ...snap.data()};
  } else {
    const nu = {
      uid: fbUser.uid,
      name: 'Guest',
      username: 'guest'+fbUser.uid.slice(0,4),
      avatar: `https://ui-avatars.com/api/?name=G&background=1877f2&color=fff`,
      online: true,
      lastSeen: serverTimestamp(),
      createdAt: serverTimestamp()
    };
    await setDoc(uref, nu);
    return nu;
  }
}

function updateMeUI(){
  $('meAvatar').src = state.user.avatar;
  $('meName').textContent = state.user.name;
  $('meUsername').textContent = '@'+state.user.username;
  $('meUid').textContent = state.user.uid;
  $('postAvatar').src = state.user.avatar;
}

function setupUI(){
  document.querySelectorAll('[data-tab]').forEach(b=>{
    b.onclick = ()=> switchTab(b.dataset.tab);
  });
  // post
  $('videoInput').onchange = e => {
    const f = e.target.files[0]; if(!f) return;
    state.pendingVideoFile = f;
    $('previewVideo').src = URL.createObjectURL(f);
    $('videoPreview').classList.remove('hidden');
  };
  $('removeVideo').onclick = ()=>{ state.pendingVideoFile=null; $('videoInput').value=''; $('videoPreview').classList.add('hidden'); };
  $('postBtn').onclick = handlePost;
  // chat
  $('addFriendBtn').onclick = addFriend;
  $('chatBack').onclick = ()=>{ $('chatView').classList.add('hidden'); $('chatListView').classList.remove('hidden'); if(state.msgsUnsub){state.msgsUnsub(); state.msgsUnsub=null;} };
  $('sendBtn').onclick = sendMessage;
  $('messageInput').onkeydown = e=>{ if(e.key==='Enter'){ e.preventDefault(); sendMessage(); } };
  // me
  $('editProfileBtn').onclick = editProfile;
  $('logoutBtn').onclick = logout;
}

function switchTab(tab){
  ['feed','post','chat','me'].forEach(t=> $(`tab-${t}`).classList.toggle('hidden', t!==tab));
  document.querySelectorAll('.nav-btn').forEach(b=>{
    const active = b.dataset.tab===tab;
    b.classList.toggle('text-[#1877f2]', active);
    b.classList.toggle('text-gray-500', !active);
    b.querySelector('i').className = b.querySelector('i').className.replace(/-fill|-line/, active ? '-fill' : '-line');
  });
  if(tab==='feed') window.scrollTo(0,0);
}

/* ---------- STORIES ---------- */
function renderStories(){
  const stories = JSON.parse(localStorage.getItem('wachat_stories')||'[]');
  $('stories').innerHTML = `
    <button class="flex flex-col items-center gap-1 min-w-[64px]" id="addStoryBtn">
      <div class="w-14 h-14 rounded-full bg-[#e7f0ff] grid place-items-center border-2 border-[#1877f2] border-dashed"><i class="ri-add-line text-[#1877f2] text-2xl"></i></div>
      <span class="text-[11px] text-gray-700">Your story</span>
    </button>
    ${stories.slice(0,12).map(s=>`
      <div class="flex flex-col items-center gap-1 min-w-[64px]">
        <img src="${s.avatar}" class="w-14 h-14 rounded-full object-cover ring-2 ring-[#1877f2] p-0.5"/>
        <span class="text-[11px] truncate w-16 text-center">${esc(s.name.split(' ')[0])}</span>
      </div>`).join('')}
  `;
  $('addStoryBtn').onclick = ()=>{
    const arr = JSON.parse(localStorage.getItem('wachat_stories')||'[]');
    arr.unshift({name:state.user.name, avatar:state.user.avatar, ts:Date.now()});
    localStorage.setItem('wachat_stories', JSON.stringify(arr.slice(0,20)));
    renderStories();
  };
}

/* ---------- FEED ---------- */
function listenPosts(){
  if(state.postsUnsub) state.postsUnsub();
  if(DEMO_MODE){
    const posts = JSON.parse(localStorage.getItem('wachat_posts')||'[]');
    renderPosts(posts);
    return;
  }
  const q = query(collection(db,'posts'), orderBy('timestamp','desc'), limit(50));
  state.postsUnsub = onSnapshot(q, snap=>{
    const posts = snap.docs.map(d=>({id:d.id, ...d.data()}));
    renderPosts(posts);
  });
}

function renderPosts(posts){
  const cont = $('postsContainer');
  const empty = $('emptyPosts');
  if(!posts.length){ cont.innerHTML=''; empty.classList.remove('hidden'); return; }
  empty.classList.add('hidden');
  cont.innerHTML = posts.map(p=>`
    <article class="bg-white rounded-2xl shadow-sm overflow-hidden">
      <div class="p-3 flex items-center gap-2.5">
        <img src="${p.avatar}" class="w-9 h-9 rounded-full object-cover"/>
        <div class="flex-1 min-w-0">
          <div class="font-semibold text-[14px] leading-tight">${esc(p.name)}</div>
          <div class="text-[12px] text-gray-500">@${esc(p.username||'')} · ${timeAgo(p.timestamp)}</div>
        </div>
        <i class="ri-more-line text-gray-400"></i>
      </div>
      ${p.text?`<p class="px-3 pb-2 text-[15px] leading-snug whitespace-pre-wrap">${esc(p.text)}</p>`:''}
      ${p.videoUrl?`<div class="bg-black"><video src="${p.videoUrl}" class="w-full max-h-[520px] video-post object-contain" autoplay muted loop playsinline></video></div>`:''}
      <div class="px-3 py-2.5 flex items-center gap-6 text-gray-600 border-t">
        <button class="flex items-center gap-1.5 hover:text-[#1877f2]" onclick="window.likePost('${p.id}')"><i class="ri-heart-3-line text-xl"></i><span class="text-sm">${p.likes||0}</span></button>
        <button class="flex items-center gap-1.5"><i class="ri-chat-1-line text-xl"></i><span class="text-sm">Reply</span></button>
        <button class="flex items-center gap-1.5 ml-auto"><i class="ri-share-forward-line text-xl"></i></button>
      </div>
    </article>
  `).join('');
}

window.likePost = async (id)=>{
  if(DEMO_MODE){
    const posts = JSON.parse(localStorage.getItem('wachat_posts')||'[]');
    const p = posts.find(x=>x.id===id); if(p){ p.likes=(p.likes||0)+1; localStorage.setItem('wachat_posts', JSON.stringify(posts)); renderPosts(posts); }
    return;
  }
  await updateDoc(doc(db,'posts',id), {likes: increment(1)});
};

async function handlePost(){
  const text = $('postText').value.trim();
  if(!text && !state.pendingVideoFile) return;
  $('postBtn').disabled = true;
  let videoUrl = '';
  try{
    if(state.pendingVideoFile){
      $('uploadProgress').classList.remove('hidden');
      videoUrl = await uploadVideo(state.pendingVideoFile);
    }
    const data = {
      userId: state.user.uid,
      name: state.user.name,
      username: state.user.username,
      avatar: state.user.avatar,
      text,
      videoUrl,
      timestamp: DEMO_MODE? Date.now() : serverTimestamp(),
      likes: 0
    };
    if(DEMO_MODE){
      const posts = JSON.parse(localStorage.getItem('wachat_posts')||'[]');
      posts.unshift({...data, id:'p'+Date.now()});
      localStorage.setItem('wachat_posts', JSON.stringify(posts));
      renderPosts(posts);
    } else {
      await addDoc(collection(db,'posts'), data);
    }
    $('postText').value=''; state.pendingVideoFile=null; $('videoPreview').classList.add('hidden'); $('uploadProgress').classList.add('hidden'); $('uploadBar').style.width='0%';
    switchTab('feed');
  }catch(e){ console.error(e); alert('Post failed'); }
  finally{ $('postBtn').disabled = false; }
}

function uploadVideo(file){
  return new Promise((resolve, reject)=>{
    if(DEMO_MODE){
      let p=0; const iv=setInterval(()=>{ p+=20; $('uploadBar').style.width=p+'%'; if(p>=100){ clearInterval(iv); resolve(URL.createObjectURL(file)); } },150);
      return;
    }
    const storageRef = ref(storage, `videos/${state.user.uid}/${Date.now()}.mp4`);
    const task = uploadBytesResumable(storageRef, file);
    task.on('state_changed', snap=>{
      const pct = (snap.bytesTransferred/snap.totalBytes)*100;
      $('uploadBar').style.width = pct+'%';
      $('uploadText').textContent = `Uploading ${Math.round(pct)}%`;
    }, reject, async ()=>{
      const url = await getDownloadURL(task.snapshot.ref);
      resolve(url);
    });
  });
}

/* ---------- CHATS & FRIENDS ---------- */
function listenFriends(){
  if(DEMO_MODE){
    const demoFriends = JSON.parse(localStorage.getItem('wachat_friends')||'[]');
    renderFriends(demoFriends);
    return;
  }
  const q = query(collection(db,'friends'), where('users','array-contains', state.user.uid));
  state.friendsUnsub = onSnapshot(q, async snap=>{
    const friends=[];
    for(const d of snap.docs){
      const data=d.data(); const oid=data.users.find(u=>u!==state.user.uid);
      const usnap = await getDoc(doc(db,'users',oid));
      if(usnap.exists()) friends.push({uid:oid, ...usnap.data()});
    }
    renderFriends(friends);
  });
}
function renderFriends(friends){
  $('friendsList').innerHTML = friends.length? friends.map(f=>`
    <button class="flex flex-col items-center gap-1 min-w-[60px]" data-fid="${f.uid}">
      <div class="relative"><img src="${f.avatar}" class="w-12 h-12 rounded-full object-cover"/><span class="absolute bottom-0 right-0 w-3 h-3 rounded-full border-2 border-white ${f.online?'bg-green-500':'bg-gray-400'}"></span></div>
      <span class="text-[11px]">${esc(f.name.split(' ')[0])}</span>
    </button>`).join('') : `<div class="text-[12px] text-gray-400 py-2">Tap "Add Friend" to start</div>`;
  document.querySelectorAll('[data-fid]').forEach(b=> b.onclick = ()=> startChatWith(b.dataset.fid));
}

function listenChats(){
  if(state.chatsUnsub) state.chatsUnsub();
  if(DEMO_MODE){
    const chats = JSON.parse(localStorage.getItem('wachat_chats')||'[]');
    renderChats(chats);
    return;
  }
  const q = query(collection(db,'chats'), where('participants','array-contains', state.user.uid));
  state.chatsUnsub = onSnapshot(q, snap=>{
    const chats = snap.docs.map(d=>({id:d.id, ...d.data()})).sort((a,b)=> (b.updatedAt?.toMillis?.()||0)-(a.updatedAt?.toMillis?.()||0));
    renderChats(chats);
  });
}

function renderChats(chats){
  const cont = $('chatsContainer');
  if(!chats.length){ cont.innerHTML = `<div class="p-8 text-center text-gray-500 text-sm">No chats yet<br> Add a friend to start messaging</div>`; return; }
  cont.innerHTML = chats.map(c=>{
    const otherId = c.participants.find(p=>p!==state.user.uid);
    const info = c.participantInfo?.[otherId] || {name:'User', avatar:'https://ui-avatars.com/api/?name=U'};
    return `
    <button class="w-full text-left px-4 py-3 hover:bg-gray-50 flex items-center gap-3" data-chat="${c.id}" data-oid="${otherId}">
      <div class="relative"><img src="${info.avatar}" class="w-12 h-12 rounded-full object-cover"/><span class="absolute bottom-0 right-0 w-3 h-3 bg-green-500 rounded-full border-2 border-white"></span></div>
      <div class="flex-1 min-w-0">
        <div class="flex items-baseline justify-between"><span class="font-medium truncate">${esc(info.name)}</span><span class="text-[11px] text-gray-400">${timeAgo(c.updatedAt)}</span></div>
        <div class="text-[13px] text-gray-500 truncate">${esc(c.lastMessage||'Start conversation')}</div>
      </div>
    </button>`;
  }).join('');
  cont.querySelectorAll('[data-chat]').forEach(b=> b.onclick = ()=> openChat(b.dataset.chat, b.dataset.oid));
}

async function addFriend(){
  const username = prompt('Enter username to add (e.g., tunde)'); if(!username) return;
  const uname = username.toLowerCase().replace('@','');
  if(DEMO_MODE){
    const fid = 'u'+Date.now();
    const friend = {uid:fid, name:username, username:uname, avatar:`https://ui-avatars.com/api/?name=${encodeURIComponent(username)}&background=0ea5e9&color=fff`, online:true};
    const friends = JSON.parse(localStorage.getItem('wachat_friends')||'[]'); friends.push(friend); localStorage.setItem('wachat_friends', JSON.stringify(friends)); renderFriends(friends);
    await startChatWith(fid, friend);
    return;
  }
  const q = query(collection(db,'users'), where('username','==',uname), limit(1));
  const snap = await getDocs(q);
  if(snap.empty) return alert('User not found');
  const f = snap.docs[0].data();
  if(f.uid===state.user.uid) return alert("That's you!");
  const fid = [state.user.uid, f.uid].sort().join('_');
  await setDoc(doc(db,'friends',fid), {users:[state.user.uid,f.uid], status:'accepted', createdAt:serverTimestamp()});
  await startChatWith(f.uid, f);
}

async function startChatWith(fid, friendData=null){
  const chatId = [state.user.uid, fid].sort().join('_');
  state.currentFriend = friendData;
  if(DEMO_MODE){
    const chats = JSON.parse(localStorage.getItem('wachat_chats')||'[]');
    if(!chats.find(c=>c.id===chatId)){
      chats.unshift({id:chatId, participants:[state.user.uid,fid], participantInfo:{[fid]:friendData}, lastMessage:'', updatedAt:Date.now()});
      localStorage.setItem('wachat_chats', JSON.stringify(chats));
    }
    openChat(chatId, fid);
    return;
  }
  const cref = doc(db,'chats',chatId);
  const csnap = await getDoc(cref);
  if(!csnap.exists()){
    const fSnap = friendData ? {data:()=>friendData} : await getDoc(doc(db,'users',fid));
    const f = fSnap.data();
    await setDoc(cref, {
      participants:[state.user.uid, fid],
      participantInfo:{
        [state.user.uid]:{name:state.user.name, username:state.user.username, avatar:state.user.avatar},
        [fid]:{name:f.name, username:f.username, avatar:f.avatar}
      },
      lastMessage:'',
      createdAt:serverTimestamp(),
      updatedAt:serverTimestamp()
    });
  }
  openChat(chatId, fid);
}

async function openChat(chatId, otherId){
  state.currentChatId = chatId;
  $('chatListView').classList.add('hidden');
  $('chatView').classList.remove('hidden'); $('chatView').classList.add('flex');
  
  let friend = state.currentFriend;
  if(!friend){
    if(DEMO_MODE){
      const friends = JSON.parse(localStorage.getItem('wachat_friends')||'[]');
      friend = friends.find(f=>f.uid===otherId) || {name:'User', avatar:'https://ui-avatars.com/api/?name=U'};
    } else {
      const fsnap = await getDoc(doc(db,'users',otherId));
      friend = fsnap.exists()? fsnap.data() : {name:'User', avatar:''};
    }
  }
  $('chatAvatar').src = friend.avatar;
  $('chatName').textContent = friend.name;
  $('chatStatus').textContent = friend.online? 'online' : 'last seen '+timeAgo(friend.lastSeen);
  $('messagesContainer').innerHTML = '';

  if(state.msgsUnsub) state.msgsUnsub();
  if(DEMO_MODE){
    const msgs = JSON.parse(localStorage.getItem('wachat_msgs_'+chatId)||'[]');
    renderMessages(msgs);
    return;
  }
  const mq = query(collection(db,'chats',chatId,'messages'), orderBy('timestamp','asc'));
  state.msgsUnsub = onSnapshot(mq, snap=>{
    const msgs = snap.docs.map(d=>({id:d.id, ...d.data()}));
    renderMessages(msgs);
  });
}

function renderMessages(msgs){
  const box = $('messagesContainer');
  box.innerHTML = msgs.map(m=>{
    const mine = m.senderId===state.user.uid;
    return `<div class="flex ${mine?'justify-end':'justify-start'}">
      <div class="max-w-[75%] px-3.5 py-2 rounded-[18px] text-[15px] leading-snug ${mine?'bg-[#1877f2] text-white rounded-br-4':'bg-white shadow-sm rounded-bl-4'}">${esc(m.text)}</div>
    </div>`;
  }).join('');
  box.scrollTop = box.scrollHeight;
}

async function sendMessage(){
  const input = $('messageInput'); const text = input.value.trim(); if(!text || !state.currentChatId) return;
  input.value='';
  if(DEMO_MODE){
    const key='wachat_msgs_'+state.currentChatId;
    const msgs = JSON.parse(localStorage.getItem(key)||'[]');
    msgs.push({senderId:state.user.uid, text, timestamp:Date.now()});
    localStorage.setItem(key, JSON.stringify(msgs));
    renderMessages(msgs);
    // update chat list
    const chats = JSON.parse(localStorage.getItem('wachat_chats')||'[]');
    const c = chats.find(x=>x.id===state.currentChatId); if(c){ c.lastMessage=text; c.updatedAt=Date.now(); localStorage.setItem('wachat_chats', JSON.stringify(chats)); renderChats(chats); }
    return;
  }
  await addDoc(collection(db,'chats',state.currentChatId,'messages'), {senderId:state.user.uid, text, timestamp:serverTimestamp()});
  await updateDoc(doc(db,'chats',state.currentChatId), {lastMessage:text, updatedAt:serverTimestamp()});
}

/* ---------- ME ---------- */
async function editProfile(){
  const name = prompt('Your display name', state.user.name); if(!name) return;
  const username = prompt('Username (lowercase, no spaces)', state.user.username); if(!username) return;
  const uname = username.toLowerCase().replace(/\s+/g,'');
  state.user.name = name; state.user.username = uname;
  state.user.avatar = `https://ui-avatars.com/api/?name=${encodeURIComponent(name)}&background=1877f2&color=fff`;
  if(DEMO_MODE){ localStorage.setItem('wachat_user', JSON.stringify(state.user)); }
  else { await updateDoc(doc(db,'users',state.user.uid), {name, username:uname, avatar:state.user.avatar}); }
  updateMeUI(); $('headerName').textContent = name;
  alert('Profile updated');
}

async function logout(){
  if(DEMO_MODE){ localStorage.clear(); location.reload(); return; }
  await updateDoc(doc(db,'users',state.user.uid), {online:false, lastSeen:serverTimestamp()});
  await signOut(auth);
  location.reload();
}

function setupTelegram(){
  if(DEMO_MODE) return;
  const cont = $('telegramContainer');
  cont.innerHTML = '';
  const s = document.createElement('script');
  s.async = true;
  s.src = 'https://telegram.org/js/telegram-widget.js?22';
  s.setAttribute('data-telegram-login', 'Wchart312_bot');
  s.setAttribute('data-size', 'large');
  s.setAttribute('data-onauth', 'onTelegramAuth(user)');
  s.setAttribute('data-request-access', 'write');
  cont.appendChild(s);
}
window.onTelegramAuth = async function(user){
  const name = [user.first_name, user.last_name].filter(Boolean).join(' ') || state.user.name;
  const uname = user.username || state.user.username;
  const avatar = user.photo_url || state.user.avatar;
  state.user.name = name; state.user.username = uname; state.user.avatar = avatar;
  if(!DEMO_MODE){
    await updateDoc(doc(db,'users',state.user.uid), {name, username:uname.toLowerCase(), avatar, telegramId:user.id});
  } else {
    localStorage.setItem('wachat_user', JSON.stringify(state.user));
  }
  updateMeUI(); $('headerName').textContent = name;
  alert('Telegram connected as @'+uname);
};

// Init
initApp();
</script>
<script>(function(){document.addEventListener("click",function(e){var a=e.target.closest("[data-product-id]");if(!a)return;e.preventDefault();var pid=a.getAttribute("data-product-id");if(pid)parent.postMessage({type:"ecto-artifact-link-click",productId:pid},"*")})})();</script>

</body></html>
