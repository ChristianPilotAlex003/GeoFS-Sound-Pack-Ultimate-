// ==UserScript==
// @name         GeoFS Multimedia Sound System(GEMOD)
// @version      1.0.0
// @description  FMOD-style rebuild of the GeoFS Sound Pack Ultimate V1.4
// @match        https://www.geo-fs.com/geofs.php*
// @match        https://beta.geo-fs.com/geofs.php
// @match        https://play.geofs.com/*
// @match        http://*/geofs.php*
// @match        https://*/geofs.php*
// @run-at       document-start
// @author       Christian Pilot Alex 003 (original packs); combined build with GGamerGGuy; Cabin Safety Audio originally by SirJackie
// @grant        none
/* eslint-disable */
// ==/UserScript==
(function(){
  'use strict';
  const _ORIG = {
    AudioContext: window.AudioContext || window.webkitAudioContext || null,
    HTMLMediaElementPlay: HTMLMediaElement.prototype.play,
    DocumentCreateElement: Document.prototype.createElement,
    ServiceWorkerRegister: navigator.serviceWorker?.register || null,
    HowlProtoPlay: window.Howl?.prototype?.play || null
  };
  // EXTRA VEHICLES COMPATIBILITY BRIDGE — GeoFS Extra Vehicles swaps aircraft
  // via aircraftRecord directly, bypassing .id, so lookups here would read
  // the previous aircraft's id until a full reload. This turns .id into a
  // live accessor that reads aircraftRecord.id (falling back to the last
  // assigned value), so no addon coordination is needed.
  (function _GE_bridgeInstanceId(){
    const PATCH_FLAG = '_GE_idBridged';
    let lastInstance = null;
    function patch(inst){
      if (!inst || inst[PATCH_FLAG]) return;
      let backing = inst.id;
      try {
        Object.defineProperty(inst, 'id', {
          configurable: true,
          enumerable: true,
          get(){
            const rec = this.aircraftRecord;
            return (rec && rec.id != null) ? rec.id : backing;
          },
          set(v){ backing = v; }
        });
        Object.defineProperty(inst, PATCH_FLAG, { value: true, configurable: true });
      } catch(e){
        // If .id is ever non-configurable, fail quietly — worst case
        // is the original full-reload-required behavior, not a crash.
      }
    }
    setInterval(() => {
      try {
        const inst = window.geofs?.aircraft?.instance;
        if (!inst) return;
        if (inst !== lastInstance) lastInstance = inst;
        patch(inst); // no-op once already patched
      } catch(e){}
    }, 250);
  })();
  window._GEPacks = window._GEPacks || {};
  function _GE_anyPackActive(){
    try {
      return Object.values(window._GEPacks).some(v => v === true);
    } catch(e){
      return false;
    }
  }
  function _GE_isTypingInTextField(e){
    try {
      const el = e?.target;
      if (!el) return false;
      const tag = String(el.tagName || '').toLowerCase();
      return tag === 'input' || tag === 'textarea' || tag === 'select' || !!el.isContentEditable;
    } catch(e){
      return false;
    }
  }
  function _GE_stopPackTransientAudio(prefix){
    try {
      const src = window[`_${prefix}_startupSrc`];
      if (src) { try { src.stop(); } catch(e){} window[`_${prefix}_startupSrc`] = null; }
    } catch(e){}
    try {
      const src = window[`_${prefix}_startupIntSrc`];
      if (src) { try { src.stop(); } catch(e){} window[`_${prefix}_startupIntSrc`] = null; }
    } catch(e){}
    try {
      const timer = window[`_${prefix}_startupViewInterval`];
      if (timer) { clearInterval(timer); window[`_${prefix}_startupViewInterval`] = null; }
    } catch(e){}
    try { window[`_${prefix}_startupPlaying`] = false; } catch(e){}
    try {
      const stopRattle = window[`${prefix.replace(/^GE90/, 'GE90')}_stopRattleLoop`];
      if (typeof stopRattle === 'function') stopRattle();
    } catch(e){}
  }
  const _GE_PACKS = [
    { code: 'a320neo', prefix: 'GE90A320NEO', ids: ['5847','2871','2865','242','4646'] },
    { code: 'a320', prefix: 'GE90A320', ids: ['5156', '2879', '3534', '3011', '5086', '2870'] },
    { code: 'a330', prefix: 'GE90A330', ids: ['244', '2856', '6012'] },
    { code: 'a339', prefix: 'GE90A339', ids: ['4631'] },
    { code: 'a380', prefix: 'GE90A380', ids: ['10'] },
    { code: 'a340', prefix: 'GE90A340', ids: ['6006', '2153', '5998'] },
    { code: 'a350', prefix: 'GE90A350', ids: ['24', '2973', '239'] },
    { code: 'b737', prefix: 'GE90B737', ids: ['4', '3054', '5203','1001'] },
    { code: 'b737max', prefix: 'GE90B737MAX', ids: ['2772', '2769'] },
    { code: 'b777', prefix: 'GE90B777', ids: ['25', '4402', '240', '1004'] }
  ];
  function _GE_zeroPackAudio(prefix){
    try {
      const ctx = window[`_${prefix}_audioCtx`];
      const now = ctx?.currentTime || 0;
      const zeroGain = node => {
        try {
          if (!node?.gain) return;
          node.gain.cancelScheduledValues(now);
          node.gain.setValueAtTime(0, now);
        } catch(e){}
      };
      zeroGain(window[`_${prefix}_master`]);
      zeroGain(window[`_${prefix}_finalMaster`]);
      const layers = window[`${prefix}_layers`];
      if (layers) {
        Object.values(layers).forEach(layer => zeroGain(layer?.gainNode));
      }
    } catch(e){}
  }
  function _GE_silenceInactivePacks(activeCode){
    _GE_PACKS.forEach(info => {
      if (info.code === activeCode) return;
      _GE_stopPackTransientAudio(info.prefix);
      _GE_zeroPackAudio(info.prefix);
    });
  }
  function _GE_activePackInfo(){
    try {
      const id = String(window.geofs?.aircraft?.instance?.id ?? '');
      if (id) {
        const byAircraftId = _GE_PACKS.find(info => info.ids.includes(id));
        if (byAircraftId) return byAircraftId;
      }
      return _GE_PACKS.find(info => window._GEPacks?.[info.code] === true) || null;
    } catch(e){
      return null;
    }
  }
  function _GE_applyPackMuteState(prefix){
    try {
      const apply = window[`_${prefix}_applyEffectiveMute`];
      if (typeof apply === 'function') apply();
    } catch(e){}
    try {
      const muted =
        !!window[`_${prefix}_muted`] ||
        !!window[`_${prefix}_userMuted`] ||
        !!window[`_${prefix}_paused`] ||
        !!window[`_${prefix}_persistentlyMuted`];
      if (muted) {
        _GE_stopPackTransientAudio(prefix);
        _GE_zeroPackAudio(prefix);
      }
    } catch(e){}
  }
  function _GE_forwardPauseToGeoFS(desiredPaused){
    try {
      const ev = new KeyboardEvent('keydown', {
        key: 'p',
        code: 'KeyP',
        keyCode: 80,
        which: 80,
        bubbles: true,
        cancelable: true
      });
      try { Object.defineProperty(ev, 'keyCode', { get: () => 80 }); } catch(e){}
      try { Object.defineProperty(ev, 'which', { get: () => 80 }); } catch(e){}
      ev._GE_forwardedGeoFSPause = true;
      ev._GE_soundHotkeyHandled = true;
      (document || window).dispatchEvent(ev);
    } catch(e){}
    try {
      const ev = new KeyboardEvent('keyup', {
        key: 'p',
        code: 'KeyP',
        keyCode: 80,
        which: 80,
        bubbles: true,
        cancelable: true
      });
      try { Object.defineProperty(ev, 'keyCode', { get: () => 80 }); } catch(e){}
      try { Object.defineProperty(ev, 'which', { get: () => 80 }); } catch(e){}
      ev._GE_forwardedGeoFSPause = true;
      ev._GE_soundHotkeyHandled = true;
      (document || window).dispatchEvent(ev);
    } catch(e){}
    setTimeout(() => {
      try {
        if (typeof window.geofs?.pause === 'boolean' && window.geofs.pause !== !!desiredPaused) {
          window.geofs.pause = !!desiredPaused;
        }
      } catch(e){}
    }, 120);
  }
  // GPWS mute wiring: pressing 's' also silences GPWS callouts/alarms immediately, independent of the persisted GPWS on/off setting.
  const _GE_GPWS_AUDIO_VARS = [
    'a2500','a2000','a1000','a500','a400','a300','a200','a100','a50','a40',
    'a30','a20','a10','aRetard','a5','stall','glideSlope','tooLowFlaps',
    'tooLowGear','apDisconnect','minimumBaro','dontSink','masterA',
    'bankAngle','overspeed','v1Callout','v1CalloutAirbus'
  ];
  window._GE_forceStopGpwsAudio = window._GE_forceStopGpwsAudio || function(skipV1){
    try {
      _GE_GPWS_AUDIO_VARS.forEach(name => {
        // The per-tick "still on the ground" housekeeping call passes skipV1=true so it
        // doesn't cut off a v1Callout that was just started this same tick (V1's trigger
        // condition requires groundContact === true, so that reset would otherwise kill
        // the sound before it's audible). Full stops (mute/pause) still stop v1Callout too.
        if (skipV1 && (name === 'v1Callout' || name === 'v1CalloutAirbus')) return;
        const a = window[name];
        if (!a) return;
        try { a.pause(); } catch(e){}
        try { a.currentTime = 0; } catch(e){}
      });
    } catch(e){}
  };
  // Note: Air Conditioning isn't stopped from here — it watches
  // window._GE_gpwsSKeyMuted / geofs.isPaused() directly, same as the APU
  // and Cabin Safety Audio.
  window._GE_gpwsSKeyMuted = window._GE_gpwsSKeyMuted || false;
  window._GE_toggleGpwsMute = window._GE_toggleGpwsMute || function(){
    window._GE_gpwsSKeyMuted = !window._GE_gpwsSKeyMuted;
    window.soundsOn = !!window._GE_gpwsOn && !window._GE_gpwsSKeyMuted;
    if (window._GE_gpwsSKeyMuted) _GE_forceStopGpwsAudio();
  };
  // SHARED: bakes a short equal-power crossfade into a decoded buffer's tail
  // so .loop=true wraps without a click. Shared by every looping buffer
  // (engine layers, buzzsaw, rattle, rain, AC hum, ambience) instead of
  // being duplicated per pack.
  window._GE_seamlessLoopBuffer = window._GE_seamlessLoopBuffer || function(ctx, srcBuf, fadeSec){
    try {
      if (!ctx || !srcBuf) return srcBuf;
      if (typeof fadeSec !== 'number' || !(fadeSec > 0)) fadeSec = 0.4;
      const sr = srcBuf.sampleRate;
      const length = srcBuf.length;
      // Adaptive cap: never let the crossfade eat more than ~25% of the
      // loop, so short clips still get the biggest smooth fade they can
      // support instead of silently falling back to a hard, un-crossfaded
      // loop.
      const maxFadeSec = (length / sr) * 0.25;
      const effectiveFadeSec = Math.min(fadeSec, maxFadeSec);
      const fadeSamples = Math.floor(effectiveFadeSec * sr);
      if (fadeSamples < 8) return srcBuf;
      const out = ctx.createBuffer(srcBuf.numberOfChannels, length, sr);
      for (let ch = 0; ch < srcBuf.numberOfChannels; ch++) {
        const inData = srcBuf.getChannelData(ch);
        const outData = out.getChannelData(ch);
        for (let i = 0; i < length; i++) outData[i] = inData[i];
        for (let i = 0; i < fadeSamples; i++) {
          // Equal-power (cosine/sine) crossfade instead of linear: linear
          // fades noticeably dip in perceived loudness mid-transition on
          // broadband/tonal loop material, which is exactly the kind of
          // "notice the loop" artifact this is meant to hide.
          const x = i / fadeSamples;
          const fadeOut = Math.cos(x * Math.PI / 2);
          const fadeIn  = Math.sin(x * Math.PI / 2);
          const tailIdx = length - fadeSamples + i;
          outData[tailIdx] = inData[tailIdx] * fadeOut + inData[i] * fadeIn;
        }
      }
      return out;
    } catch(e){
      return srcBuf;
    }
  };
  // A350 HUD toggle sound: layers a click on the native 'z' HUD toggle when A350 is active; purely additive, never blocks the keystroke.
  window._GE_a350HudSound = window._GE_a350HudSound ||
    new Audio('https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A350-Sound-Repository/main/A350HUD.wav');
  try {
    // Tagged the same way GPWS callouts are (see the play() wrapper below)
    // so this isn't force-muted while the A350 pack is active.
    window._GE_a350HudSound.dataset.geofsAllow = 'true';
    window._GE_a350HudSound.dataset.ge90Allow = 'true';
  } catch(e){}
  if (!window._GE_a350HudKeyInstalled) {
    window._GE_a350HudKeyInstalled = true;
    window.addEventListener('keydown', e => {
      try {
        if (_GE_isTypingInTextField(e) || e.repeat) return;
        if (e.ctrlKey || e.metaKey || e.altKey) return;
        if (String(e.key || '').toLowerCase() !== 'z') return;
        const info = _GE_activePackInfo();
        if (!info || info.code !== 'a350') return;
        // Respect the same S-key mute the rest of the pack uses — if the
        // A350 pack is muted (or persistently muted), stay silent instead
        // of still clicking on every HUD toggle.
        const prefix = info.prefix;
        const isMuted =
          !!window[`_${prefix}_muted`] ||
          !!window[`_${prefix}_userMuted`] ||
          !!window[`_${prefix}_persistentlyMuted`];
        if (isMuted) return;
        const snd = window._GE_a350HudSound;
        try { snd.currentTime = 0; } catch(err){}
        snd.play().catch(()=>{});
      } catch(err){
        console.warn('[GE90 Ultimate] A350 HUD sound failed', err);
      }
    }, true);
  }
  if (!window._GE_sharedSoundHotkeysInstalled) {
    window._GE_sharedSoundHotkeysInstalled = true;
    window.addEventListener('keydown', e => {
      try {
        if (e._GE_forwardedGeoFSPause) return;
        if (_GE_isTypingInTextField(e) || e.repeat) return;
        const key = String(e.key || '').toLowerCase();
        if (key !== 's' && key !== 'p') return;
        // GPWS mute is independent of which (if any) engine sound pack is
        // active, so it toggles here unconditionally rather than inside
        // the info-gated block below.
        if (key === 's') _GE_toggleGpwsMute();
        const info = _GE_activePackInfo();
        if (!info) {
          _GE_silenceInactivePacks(null);
          setTimeout(() => _GE_silenceInactivePacks(null), 80);
          if (key === 'p') {
            setTimeout(() => _GE_silenceInactivePacks(null), 350);
            setTimeout(() => _GE_silenceInactivePacks(null), 1000);
          }
          return;
        }
        e._GE_soundHotkeyHandled = true;
        try { e.preventDefault(); } catch(err){}
        try { e.stopImmediatePropagation(); } catch(err){}
        _GE_silenceInactivePacks(info.code);
        const fnName = key === 's' ? `_${info.prefix}_toggleMuteGlobal` : `_${info.prefix}_togglePauseGlobal`;
        const fn = window[fnName];
        if (typeof fn === 'function') fn();
        if (key === 'p') _GE_forwardPauseToGeoFS(!!window[`_${info.prefix}_paused`]);
        _GE_applyPackMuteState(info.prefix);
        setTimeout(() => {
          _GE_silenceInactivePacks(info.code);
          _GE_applyPackMuteState(info.prefix);
        }, 80);
        if (key === 'p') {
          setTimeout(() => {
            _GE_silenceInactivePacks(info.code);
            _GE_applyPackMuteState(info.prefix);
          }, 350);
          setTimeout(() => {
            _GE_silenceInactivePacks(info.code);
            _GE_applyPackMuteState(info.prefix);
          }, 1000);
        }
      } catch(err){
        console.warn('[GE90 Ultimate] shared hotkey routing failed', err);
      }
    }, true);
  }
  // HTMLMediaElement wrapper (shared): mutes any non-GeoFS-native audio element while a pack is active, except elements tagged _GE90_TAG.
  try {
    HTMLMediaElement.prototype.play = function(){
      try {
        if (!_GE_anyPackActive()) return _ORIG.HTMLMediaElementPlay.apply(this, arguments);
        if (this._GE90_TAG) return _ORIG.HTMLMediaElementPlay.apply(this, arguments);
        if (this.dataset?.geofsAllow === 'true' || this.dataset?.ge90Allow === 'true' || this.dataset?.gpwsAllow === 'true')
          return _ORIG.HTMLMediaElementPlay.apply(this, arguments);
        const src = (this.currentSrc || this.src || '').toString();
        // 'tylerbmusic.github.io' explicitly allowlisted: that's where the
        // GeoFS GPWS callouts userscript hosts its audio files. This keeps
        // GPWS callouts compatible with this pack even if the "geofs"
        // substring match below ever stops matching that host's URLs.
        if (src.includes('geo-fs.com') || src.includes('geofs') || src.includes(location.hostname) || src.includes('tylerbmusic.github.io')) {
          this.dataset.geofsAllow = 'true';
          return _ORIG.HTMLMediaElementPlay.apply(this, arguments);
        }
        this.pause();
        this.muted = true;
        this.volume = 0;
        this._GE90_FORCED_MUTE_BY_PACK = true;
        return Promise.resolve();
      } catch(e){
        return Promise.resolve();
      }
    };
  } catch(e){}
  // -------------------------
  // createElement wrapper (shared)
  // -------------------------
  try {
    Document.prototype.createElement = function(tagName, options){
      const el = _ORIG.DocumentCreateElement.call(this, tagName, options);
      if (['audio','video'].includes(String(tagName).toLowerCase()))
        el._GE90_BLOCKED = false;
      return el;
    };
  } catch(e){}
  // -------------------------
  // AudioContext fencing (shared)
  // -------------------------
  (function wrapAC(){
    const AC = _ORIG.AudioContext;
    if (!AC) return;
    const proto = AC.prototype;
    if (proto.createBufferSource && !proto.createBufferSource._GE90_wrapped) {
      const orig = proto.createBufferSource;
      proto.createBufferSource = function(){
        const src = orig.apply(this, arguments);
        const origConnect = src.connect;
        src.connect = function(node, ...rest){
          try {
            if (!_GE_anyPackActive()) return origConnect.apply(this, arguments);
            if (src._GE90_TAG) return origConnect.apply(this, arguments);
            if (node?._GE90_TAG) return origConnect.apply(this, arguments);
            const ctx = this.context || new AC();
            const g = ctx.createGain();
            g.gain.value = 0;
            g.connect(ctx.destination);
            return origConnect.call(this, g, ...rest);
          } catch(e){
            return origConnect.apply(this, arguments);
          }
        };
        proto.createBufferSource._GE90_wrapped = true;
        return src;
      };
    }
  })();
  // -------------------------
  // Howler fencing (shared) — gated by any-pack-active, never
  // unconditionally muted, so aircraft without a pack keep default sound.
  // -------------------------
  try {
    if (window.Howl?.prototype) {
      const orig = _ORIG.HowlProtoPlay || window.Howl.prototype.play;
      window.Howl.prototype.play = function(){
        if (!_GE_anyPackActive()) return orig.apply(this, arguments);
        if (this._GE90_TAG) return orig.apply(this, arguments);
        return null;
      };
    }
  } catch(e){}
  // -------------------------
  // Block service worker (shared)
  // -------------------------
  try {
    if (navigator.serviceWorker?.register) {
      navigator.serviceWorker.register = () =>
        Promise.reject(new Error('blocked by GE90 userscript'));
    }
  } catch(e){}
  // GEFMOD APU INTERLOCK — engine start requires the APU to be ON.
  // Shared by every pack. window._GE_apuState must be fully 'on' (not
  // 'starting') to unlock engine start; only off->on transitions are
  // blocked, so switching the APU off never stops a running engine.
  // Exempt: airborne restarts, and builds with no APU panel (state undefined).
  function _GE_apuAllowsEngineStart(){
    try {
      const s = window._GE_apuState;
      if (s === undefined || s === 'on') return true;
      const inst = window.geofs?.aircraft?.instance;
      if (inst && inst.groundContact === false) return true;
      return false;
    } catch(e){ return true; }
  }
  // Warning popup: deliberately styled to match the S/P mute indicator
  // (showMuteIndicator) — same top-right placement, box, font and 3 s
  // fade — in the red "warning" colour used for PERSISTENT MUTE.
  let _GE_apuNoticeAt = 0;
  function _GE_apuInterlockNotice(){
    try {
      const now = performance.now();
      if (now - _GE_apuNoticeAt < 400) return;
      _GE_apuNoticeAt = now;
      const existing = document.getElementById('geofs-apu-interlock-indicator');
      if (existing) existing.remove();
      const indicator = document.createElement('div');
      indicator.id = 'geofs-apu-interlock-indicator';
      indicator.textContent = '\u26D4 APU REQUIRED FOR ENGINE START';
      indicator.style.cssText = `
        position: fixed;
        top: 20px;
        right: 20px;
        background: rgba(220, 53, 69, 0.9);
        color: white;
        padding: 12px 20px;
        border-radius: 8px;
        font-size: 14px;
        font-weight: bold;
        z-index: 10000;
        transition: opacity 0.3s;
        box-shadow: 0 4px 12px rgba(0,0,0,0.3);
        font-family: Arial, sans-serif;
      `;
      document.body.appendChild(indicator);
      setTimeout(() => {
        if (indicator.parentNode) {
          indicator.style.opacity = '0';
          setTimeout(() => {
            if (indicator.parentNode) {
              indicator.parentNode.removeChild(indicator);
            }
          }, 300);
        }
      }, 3000);
    } catch(e){}
  }
  // Wraps the shared Aircraft.prototype.startEngines() (plural) — the real
  // entry point GeoFS's key handler calls (confirmed via console tracing).
  // Patched once on the prototype, so every aircraft/pack is covered.
  // Refusing here means engine.on never flips, so there's nothing to catch
  // or wind down afterward.
  function _GE_installApuStartEnginesGuard(){
    try {
      const proto = window.geofs?.aircraft?.Aircraft?.prototype;
      if (!proto || typeof proto.startEngines !== 'function') return false;
      if (proto.startEngines._GE_apuGuarded) return true; // already patched
      const origStartEngines = proto.startEngines;
      const wrapped = function(){
        if (!_GE_apuAllowsEngineStart()) { _GE_apuInterlockNotice(); return; }
        return origStartEngines.apply(this, arguments);
      };
      wrapped._GE_apuGuarded = true;
      proto.startEngines = wrapped;
      return true;
    } catch(e){
      return false;
    }
  }
  // geofs.aircraft.Aircraft may not exist yet this early (document-start),
  // so retry until it does, then stop polling. Installed once, globally,
  // independent of any pack's lifecycle.
  (function _GE_waitForApuStartEnginesGuard(){
    if (_GE_installApuStartEnginesGuard()) return;
    const poll = setInterval(() => {
      if (_GE_installApuStartEnginesGuard()) clearInterval(poll);
    }, 250);
  })();
  // =====================================================================
  // GEFMOD RUNTIME — the pack factory. Every aircraft runs this same code;
  // everything that differed between the ten original modules is now a
  // bank object (B).
  // =====================================================================
  function createPack(B){
    const CODE = B.code;
    const SFX  = B.suffix;
    const NP   = 'GE90' + SFX;        // window.GE90<SFX>_*
    const IP   = '_GE90' + SFX;       // window._GE90<SFX>_*
  // PACK FACTORY — one implementation, driven entirely by a bank
  // =====================================================================
  (function(){
  // -------------------------
  // Tunables / Save originals
  // -------------------------
  window[IP+'_active']  = false;
  window[IP+'_started'] = false;
(function initGE90(){
    try {
      // -------------------------
      // Global mute/pause state + controls — defined here so S/P work
      // correctly even before a target aircraft has been detected.
      // -------------------------
      window[IP+'_userMuted'] = window[IP+'_userMuted'] || false;
      window[IP+'_paused']    = window[IP+'_paused']    || false;
      window[IP+'_persistentlyMuted'] = window[IP+'_userMuted'];
      // Note: native GeoFS audio mute/unmute is handled entirely by the
      // shared cross-pack arbiter (driven by window._GEPacks), not here.
      // Toggling this pack's own mute/pause only affects its own layers.
      window[IP+'_toggleMuteGlobal'] = function(){
        if (typeof window[IP+'_toggleMuteFull'] === 'function') {
          return window[IP+'_toggleMuteFull']();
        }
        window[IP+'_userMuted'] = !window[IP+'_userMuted'];
        window[IP+'_persistentlyMuted'] = window[IP+'_userMuted'];
      };
      window[IP+'_togglePauseGlobal'] = function(){
        if (typeof window[IP+'_togglePauseFull'] === 'function') {
          return window[IP+'_togglePauseFull']();
        }
        window[IP+'_paused'] = !window[IP+'_paused'];
          window[IP+'_userPaused'] = window[IP+'_paused'];
      };
      window.addEventListener('keydown', e => {
        try {
          if (e._GE_soundHotkeyHandled || _GE_isTypingInTextField(e) || e.repeat || !window[IP+'_active']) return;
          if (e.key === 's' || e.key === 'S') {
            window[IP+'_toggleMuteGlobal']();
          } else if (e.key === 'p' || e.key === 'P') {
            window[IP+'_togglePauseGlobal']();
          }
        } catch(e){}
      });
      const start = () => {
        try {
          const AC = _ORIG.AudioContext || window.AudioContext || window.webkitAudioContext;
          if (!AC) return console.warn('[GE90] no AudioContext');
          const geCtx = new AC();
          geCtx._GE90_owned = true;
          geCtx[IP+'_owned'] = true;
          window[IP+'_audioCtx'] = geCtx;
          const master = geCtx.createGain();
          master.gain.setValueAtTime(1, geCtx.currentTime);
          master._GE90_TAG = true;
          master.connect(geCtx.destination);
          window[IP+'_master'] = master;
          window[IP+'_master_boost'] = window[IP+'_master_boost'] || B.mix.masterBoost;
          let startTime = performance.now();
          window[IP+'_resetStartTime'] = () => { startTime = performance.now(); };
          window[IP+'_startupSrc'] = null;
            let _engineStateArmed = false;
          let _engineStateArmTicks = 0;
          const ENGINE_ARM_TICKS = 12; // ~12 poll ticks (~960ms) of silence before reacting
          async function fetchDecode(url){
            const r = await fetch(url, { mode: 'cors' });
            const ab = await r.arrayBuffer();
            return await geCtx.decodeAudioData(ab);
          }
          const FILES = B.layers;
          async function createLayer(key){
            try {
              const url = FILES[key];
              const buf = await fetchDecode(url);
              const src = geCtx.createBufferSource();
              src.buffer = window._GE_seamlessLoopBuffer(geCtx, buf, 0.4);
              src.loop = true;
              src._GE90_TAG = true;
              const g = geCtx.createGain();
              g.gain.setValueAtTime(key === 'idle' ? 1 : 0, geCtx.currentTime);
              g._GE90_TAG = true;
              const lp = geCtx.createBiquadFilter();
              lp.type = 'lowpass';
              lp.frequency.value = (B.layerLpf && B.layerLpf[key] != null) ? B.layerLpf[key] : 20000;
              if (B.engine.idleUnfiltered && key === 'idle') lp.Q.value = 0.7;
              src.connect(g);
              g.connect(lp);
              if (B.engine.idleUnfiltered && key === 'idle' && window[IP+'_acousticChainInput']) {
                lp.connect(window[IP+'_acousticChainInput']);
              } else {
                lp.connect(master);
              }
              src.start(0);
              window[NP+'_layers'] = window[NP+'_layers'] || {};
              window[NP+'_layers'][key] = { src, gainNode: g, filter: lp };
              return true;
            } catch(e){
              console.warn('[GE90] createLayer failed', key, e);
              return false;
            }
          }
async function createBuzzsawLayer(){
    try {
        const buf = await fetchDecode(B.assets.buzzsaw);
        const src = geCtx.createBufferSource();
        src.buffer = window._GE_seamlessLoopBuffer(geCtx, buf, 0.3);
        src.loop = true;
        src._GE90_TAG = true;
        const g = geCtx.createGain();
        g.gain.setValueAtTime(0, geCtx.currentTime);
        g._GE90_TAG = true;
        // optional simple LPF for buzzsaw character
        const lp = geCtx.createBiquadFilter();
        lp.type = "lowpass";
        lp.frequency.value = B.buzzsaw.lpf;
        lp.Q.value = 0.7;
        src.connect(g);
        g.connect(lp);
        lp.connect(master);   // same as idle/n1/toga
        src.start(0);
        window[NP+'_layers'] = window[NP+'_layers'] || {};
        window[NP+'_layers']["buzzsaw"] = { src, gainNode: g, filter: lp };
    } catch(e){
        console.warn("[GE90] Buzzsaw layer failed", e);
    }
}
// View-dependent acoustic EQ chain: post-mix EQ between engine layers and master, per camera view.
(function setupAcousticChain(){
    const PROFILES = B.profiles;
    PROFILES.jumpseat = PROFILES.cockpit;
    const TRANSITION_TC = 0.35; // seconds — smooth morph between profiles
    const POLL_MS       = 120;
    let chainReady = false;
    let chainGain, chainLowShelf, chainPeak, chainHighShelf, chainLpf;
    function buildChain(){
        if (chainReady) return;
        chainGain = geCtx.createGain();
        chainGain.gain.value = 1.0;
        chainGain._GE90_TAG = true;
        chainLowShelf = geCtx.createBiquadFilter();
        chainLowShelf.type = 'lowshelf';
        chainLowShelf.frequency.value = 200;
        chainLowShelf.gain.value = 0;
        chainLowShelf._GE90_TAG = true;
        chainPeak = geCtx.createBiquadFilter();
        chainPeak.type = 'peaking';
        chainPeak.frequency.value = 1000;
        chainPeak.gain.value = 0;
        chainPeak.Q.value = 1.0;
        chainPeak._GE90_TAG = true;
        chainHighShelf = geCtx.createBiquadFilter();
        chainHighShelf.type = 'highshelf';
        chainHighShelf.frequency.value = 3000;
        chainHighShelf.gain.value = 0;
        chainHighShelf._GE90_TAG = true;
        chainLpf = geCtx.createBiquadFilter();
        chainLpf.type = 'lowpass';
        chainLpf.frequency.value = 20000;
        chainLpf.Q.value = 0.7;
        chainLpf._GE90_TAG = true;
        // Internal chain: chainGain → lowShelf → peak → highShelf → lpf → master
        chainGain.connect(chainLowShelf);
        chainLowShelf.connect(chainPeak);
        chainPeak.connect(chainHighShelf);
        chainHighShelf.connect(chainLpf);
        chainLpf.connect(master);
        // Rewire all engine layers: layer.filter → chainGain (instead of master)
        ['idle','n1','toga','buzzsaw'].forEach(k => {
            const layer = window[NP+'_layers']?.[k];
            if (!layer?.filter || layer._GE90_proximityOwned) return;
            try { layer.filter.disconnect(master); } catch(e){}
            layer.filter.connect(chainGain);
        });
        // Expose chain input for proximity module
        window[IP+'_acousticChainInput'] = chainGain;
        chainReady = true;
    }
    function ensureChainWired(){
        const layers = window[NP+'_layers'];
        if (!layers?.idle || !layers?.n1 || !layers?.toga) return false;
        if (!chainReady){
            buildChain();
            return true;
        }
        // Respawn safety — re-check wiring on every tick
        ['idle','n1','toga','buzzsaw'].forEach(k => {
            const layer = layers[k];
            if (!layer?.filter || layer._GE90_proximityOwned) return;
            try { layer.filter.disconnect(master); } catch(e){}
            try { layer.filter.connect(chainGain); } catch(e){}
        });
        return true;
    }
    function applyProfile(p){
        const now = geCtx.currentTime;
        const tc  = TRANSITION_TC;
        chainGain.gain.setTargetAtTime(p.gain, now, tc);
        chainLowShelf.frequency.setTargetAtTime(p.lowShelfFreq, now, tc);
        chainLowShelf.gain.setTargetAtTime(p.lowShelfGain, now, tc);
        chainPeak.frequency.setTargetAtTime(p.peakFreq, now, tc);
        chainPeak.gain.setTargetAtTime(p.peakGain, now, tc);
        chainPeak.Q.setTargetAtTime(p.peakQ, now, tc);
        chainHighShelf.frequency.setTargetAtTime(p.highShelfFreq, now, tc);
        chainHighShelf.gain.setTargetAtTime(p.highShelfGain, now, tc);
        chainLpf.frequency.setTargetAtTime(p.lpfFreq, now, tc);
    }
    // Directional exterior blend: bearing from aircraft to camera vs. nose heading sets brighter/duller tone.
    function lerp(a, b, t){ return a + (b - a) * t; }
    function _dirLlaToEcef(lat, lon, alt){
        const a = 6378137;
        const e2 = 0.00669437999014;
        const latR = lat * Math.PI/180;
        const lonR = lon * Math.PI/180;
        const N = a / Math.sqrt(1 - e2 * Math.sin(latR)**2);
        const x = (N + alt) * Math.cos(latR) * Math.cos(lonR);
        const y = (N + alt) * Math.cos(latR) * Math.sin(lonR);
        const z = (N * (1-e2) + alt) * Math.sin(latR);
        return { x, y, z };
    }
    // 1 = camera dead ahead of the nose, 0 = dead behind the tail,
    // ~0.5 off to either side. Returns null if data isn't ready yet.
    function computeFrontAmount(){
        try {
            const camPos = window.geofs?.camera?.cam?.position;
            const lla = window.geofs?.aircraft?.instance?.llaLocation;
            const heading = window.geofs?.animation?.values?.heading360;
            if (!camPos || !lla || typeof heading !== 'number') return null;
            const lat = lla[0], lon = lla[1], alt = lla[2];
            const acEcef = _dirLlaToEcef(lat, lon, alt);
            const dx = camPos.x - acEcef.x;
            const dy = camPos.y - acEcef.y;
            const dz = camPos.z - acEcef.z;
            const latR = lat * Math.PI/180;
            const lonR = lon * Math.PI/180;
            const east  = -Math.sin(lonR)*dx + Math.cos(lonR)*dy;
            const north = -Math.sin(latR)*Math.cos(lonR)*dx - Math.sin(latR)*Math.sin(lonR)*dy + Math.cos(latR)*dz;
            if (Math.abs(east) < 1e-6 && Math.abs(north) < 1e-6) return null;
            let bearingToCam = Math.atan2(east, north) * 180 / Math.PI;
            if (bearingToCam < 0) bearingToCam += 360;
            let rel = bearingToCam - heading;
            rel = ((rel + 180) % 360 + 360) % 360 - 180; // normalize to (-180,180]
            return (Math.cos(rel * Math.PI/180) + 1) / 2;
        } catch(e){
            return null;
        }
    }
    function blendExteriorProfile(t){
        const PA = PROFILES.exteriorBehind, PB = PROFILES.exteriorFront;
        return {
            gain:          lerp(PA.gain, PB.gain, t),
            lowShelfFreq:  lerp(PA.lowShelfFreq, PB.lowShelfFreq, t),
            lowShelfGain:  lerp(PA.lowShelfGain, PB.lowShelfGain, t),
            peakFreq:      lerp(PA.peakFreq, PB.peakFreq, t),
            peakGain:      lerp(PA.peakGain, PB.peakGain, t),
            peakQ:         lerp(PA.peakQ, PB.peakQ, t),
            highShelfFreq: lerp(PA.highShelfFreq, PB.highShelfFreq, t),
            highShelfGain: lerp(PA.highShelfGain, PB.highShelfGain, t),
            lpfFreq:       lerp(PA.lpfFreq, PB.lpfFreq, t),
        };
    }
    let lastView = null;
    let smoothedFrontAmount = 0.5;
    const DIR_SMOOTH_ALPHA = 0.12; // smooths camera-orbit direction changes
    setInterval(() => {
        try {
            if (!ensureChainWired()) return;
            const view = (typeof getViewType === 'function')
                ? getViewType()
                : (window[IP+'_viewType'] || 'exterior');
            // Exterior (everything besides cockpit/jumpseat/wingEngine/wing2)
            // gets a continuously-updated front/behind blend instead of a
            // single static profile, since the camera can orbit freely.
            if (view === 'exterior') {
                const rawFront = computeFrontAmount();
                if (typeof rawFront === 'number') {
                    smoothedFrontAmount += (rawFront - smoothedFrontAmount) * DIR_SMOOTH_ALPHA;
                }
                lastView = view;
                applyProfile(blendExteriorProfile(smoothedFrontAmount));
                return;
            }
            if (view === lastView) return;
            lastView = view;
            const profile = PROFILES[view] || PROFILES.exterior;
            applyProfile(profile);
        } catch(e){}
    }, POLL_MS);
})();
          // -------------------------
          // Exterior startup clip: a plain URL for most banks, or a
          // function when a bank ships different clips per variant
          // (the A350-1000 has its own startup sound).
          function resolveStartupExt(){
            const v = B.assets.startupExt;
            return typeof v === 'function' ? v() : v;
          }
          // One-shot playback (startup/shutdown)
          // -------------------------
          async function playOneShot(url, gain = 1.0){
            try {
              if (performance.now() - startTime < B.engine.startupSuppressMs) return;
              const buf = await fetchDecode(url);
              const src = geCtx.createBufferSource();
              const g = geCtx.createGain();
              const lpf = geCtx.createBiquadFilter();
              lpf.type = "lowpass";
              lpf.Q.value = 0.7;
              lpf._GE90_TAG = true;
              const proxVol   = window[IP+'_proximityVolMult']   || 1.0;
              const proxLpf   = window[IP+'_proximityLpfHz']     || 20000;
              const proxPitch = window[IP+'_proximityPitchMult'] || 1.0;
              g.gain.value = gain * proxVol;
              lpf.frequency.value = proxLpf;
              src.playbackRate.value = proxPitch;
              src.buffer = buf;
              src._GE90_TAG = true;
              g._GE90_TAG = true;
              src.connect(g);
              g.connect(lpf);
              // Same low-pass + volume-reducer chain the engine layers are
              // sent through (chainGain -> EQ -> chainLpf -> master), not a
              // direct connection to master, so startup/shutdown one-shots
              // are muffled/attenuated per-view exactly like the layers.
              const chainTarget = window[IP+'_acousticChainInput'] || master;
              lpf.connect(chainTarget);
              src.start(0);
              if (url === resolveStartupExt()) {
                try {
                  if (window[IP+'_startupSrc']) {
                    try { window[IP+'_startupSrc'].stop(); } catch(e){}
                  }
                } catch(e){}
                window[IP+'_startupSrc'] = src;
              }
            } catch(e){
              console.warn('[GE90] one-shot failed', url, e);
            }
          }
// ------------------------
// Triggers Gear Sound
// ------------------------
async function playGearClick() {
    try {
        const buf = await fetchDecode(B.assets.gearClick);
        const src = geCtx.createBufferSource();
        const g = geCtx.createGain();
        src.buffer = buf;
        src._GE90_TAG = true;
        g._GE90_TAG = true;
        g.gain.value = 0.80; // Increased from 0.60 (gear click louder)
        src.connect(g);
        g.connect(master);
        src.start(0);
    } catch(e){
        console.warn("[GE90] gear click failed", e);
    }
}
window[NP+'_playGearClick'] = playGearClick;
// ------------------------------------------------------
// Cockpit Tray Table Animation Sound (cockpit + jumpseat only)
// ------------------------------------------------------
(function(){
    let lastTrayState = null;
    async function playTraySound() {
        try {
            const r = await fetch(B.assets.tray);
            const ab = await r.arrayBuffer();
            const buf = await geCtx.decodeAudioData(ab);
            const src = geCtx.createBufferSource();
            const g = geCtx.createGain();
            src.buffer = buf;
            src._GE90_TAG = true;
            g._GE90_TAG = true;
            g.gain.value = 0.75; // adjust volume here
            src.connect(g);
            g.connect(master);
            src.start(0);
        } catch(e){
            console.warn("[GE90] tray table sound failed", e);
        }
    }
    setInterval(() => {
        try {
            const anim = geofs?.animation?.values;
            if (!anim) return;
            // The real animation variable
            const trayState = anim.optionalAnimatedPartTarget;
            if (trayState == null) return;
            if (lastTrayState === null) {
                lastTrayState = trayState;
                return;
            }
            // Detect toggle (0 → 1 or 1 → 0)
            if (trayState !== lastTrayState) {
                lastTrayState = trayState;
                // View check
                const view = (typeof getViewType === "function")
                    ? getViewType()
                    : (window[IP+'_viewType'] || "exterior");
                if (view === "cockpit" || view === "jumpseat") {
                    playTraySound();
                }
            }
        } catch(e){}
    }, 120);
})();
// ---------------------------------------------
// Dual startup sound: interior + exterior
// ---------------------------------------------
async function playStartupDual() {
  try {
    if (performance.now() - startTime < B.engine.startupSuppressMs) return;
    const view = (typeof getViewType === "function")
      ? getViewType()
      : (window[IP+'_viewType'] || "exterior");
    // Load both buffers
    const bufExt = await fetchDecode(resolveStartupExt());
    const bufInt = await fetchDecode(B.assets.startupInt);
    // Create nodes
    const srcExt = geCtx.createBufferSource();
    const srcInt = geCtx.createBufferSource();
    srcExt.buffer = bufExt;
    srcInt.buffer = bufInt;
    srcExt._GE90_TAG = true;
    srcInt._GE90_TAG = true;
    const gExt = geCtx.createGain();
    const gInt = geCtx.createGain();
    gExt._GE90_TAG = true;
    gInt._GE90_TAG = true;
    const lpfExt = geCtx.createBiquadFilter();
    lpfExt.type = "lowpass";
    lpfExt.Q.value = 0.7;
    lpfExt._GE90_TAG = true;
    const lpfInt = geCtx.createBiquadFilter();
    lpfInt.type = "lowpass";
    lpfInt.Q.value = 0.7;
    lpfInt._GE90_TAG = true;
    // View‑based gain logic
    let extGain = 0.22;
    let intGain = 0.22;
    if (view === "cockpit" || view === "jumpseat") {
      extGain = 0.09;
      intGain = 0.60;
    }
    else if (view === "wingEngine") {
      extGain = 0.10;  // exterior still audible — you're right next to the engine
      intGain = 0.55;  // strong interior layer too
    }
    else if (view === "wing2") {
      extGain = 0.10;
      intGain = 0.58;
    }
    else { // exterior
      extGain = 0.60;
      intGain = 0.10;
    }
// Re-check suppression after async fetches — fetching buffers can take
    // long enough that the suppression window expires mid-fetch
    if (performance.now() - startTime < B.engine.startupSuppressMs) return;
    if (!_engineStateArmed) return;
    const _proxVol0   = window[IP+'_proximityVolMult']   || 1.0;
    const _proxLpf0   = window[IP+'_proximityLpfHz']     || 20000;
    const _proxPitch0 = window[IP+'_proximityPitchMult'] || 1.0;
    gExt.gain.value = extGain * _proxVol0;
    gInt.gain.value = intGain * _proxVol0;
    lpfExt.frequency.value = _proxLpf0;
    lpfInt.frequency.value = _proxLpf0;
    srcExt.playbackRate.value = _proxPitch0;
    srcInt.playbackRate.value = _proxPitch0;
    // Connect (proximity-driven lowpass fades/darkens/pitches startup with camera distance too)
    srcExt.connect(gExt);
    srcInt.connect(gInt);
    gExt.connect(lpfExt);
    gInt.connect(lpfInt);
    // Same low-pass + volume-reducer chain the engine layers are sent
    // through, so startup sounds match layer loudness/muffling per view
    // instead of bypassing it straight to master.
    const chainTarget = window[IP+'_acousticChainInput'] || master;
    lpfExt.connect(chainTarget);
    lpfInt.connect(chainTarget);
    // Start both
    srcExt.start(0);
    srcInt.start(0);
  // Track both sources globally
window[IP+'_startupSrc'] = srcExt;
window[IP+'_startupIntSrc'] = srcInt;
// ------------------------------------------------------
// Dynamic view-based gain updater during startup
// ------------------------------------------------------
window[IP+'_startupViewInterval'] = setInterval(() => {
    try {
        // If both sounds are gone, stop updating
    if (!window[IP+'_startupSrc'] && !window[IP+'_startupIntSrc']) {
            clearInterval(window[IP+'_startupViewInterval']);
            window[IP+'_startupViewInterval'] = null;
           window[IP+'_startupPlaying'] = false;
            // Let the RPM poll loop take over naturally on next tick
            // Force idle layer audible as a safety net in case norm=0 at idle
            try {
                const idleLayer = window[NP+'_layers']?.idle;
                if (idleLayer?.gainNode) {
                    idleLayer.gainNode.gain.cancelScheduledValues(geCtx.currentTime);
                    idleLayer.gainNode.gain.setValueAtTime(0.001, geCtx.currentTime);
                    idleLayer.gainNode.gain.setTargetAtTime(0.35, geCtx.currentTime, 0.8);
                }
            } catch(e){}
        }
        const view = (typeof getViewType === "function")
            ? getViewType()
            : (window[IP+'_viewType'] || "exterior");
        let extGain = 0.22;
        let intGain = 0.22;
        if (view === "cockpit" || view === "jumpseat") {
            extGain = 0.08;
            intGain = 0.55;
        }
        else if (view === "wingEngine") {
            extGain = 0.10;
            intGain = 0.55;
        }
        else if (view === "wing2") {
            extGain = 0.10;
            intGain = 0.60;
        }
        else { // exterior
            extGain = 0.55;
            intGain = 0.02;
        }
        const _proxVol   = window[IP+'_proximityVolMult']   || 1.0;
        const _proxLpf   = window[IP+'_proximityLpfHz']     || 20000;
        const _proxPitch = window[IP+'_proximityPitchMult'] || 1.0;
        if (gExt) gExt.gain.setTargetAtTime(extGain * _proxVol, geCtx.currentTime, 0.05);
        if (gInt) gInt.gain.setTargetAtTime(intGain * _proxVol, geCtx.currentTime, 0.05);
        if (lpfExt) lpfExt.frequency.setTargetAtTime(_proxLpf, geCtx.currentTime, 0.15);
        if (lpfInt) lpfInt.frequency.setTargetAtTime(_proxLpf, geCtx.currentTime, 0.15);
        if (srcExt?.playbackRate) srcExt.playbackRate.setTargetAtTime(_proxPitch, geCtx.currentTime, 0.15);
        if (srcInt?.playbackRate) srcInt.playbackRate.setTargetAtTime(_proxPitch, geCtx.currentTime, 0.15);
    } catch(e){}
}, 120);
  } catch (e) {
    console.warn("[GE90] dual startup failed", e);
  }
}
          // -------------------------
          // Throttle curve
          // -------------------------
          function computeTargets(t){
            const clamp = v => Math.max(0, Math.min(1, v));
            const idleCut = B.curve.idleCut;
            const n1Start = B.curve.n1Start, n1Full = B.curve.n1Full;
            const togaStart = B.curve.togaStart;
            const MID_CENTER = B.curve.midCenter, MID_WIDTH = B.curve.midWidth, MID_GAIN = B.curve.midGain;
            const MIN_LAYER_FLOOR = B.curve.floor;
            const rawIdleLinear = clamp(1 - (t / idleCut));
            const idleRaw = Math.pow(rawIdleLinear, B.curve.idlePower);
            let n1Raw = 0;
            if (t > n1Start)
              n1Raw = t <= n1Full ? (t - n1Start) / (n1Full - n1Start) : 1;
            const togaRaw = t <= togaStart ? 0 : Math.pow((t - togaStart) / (1 - togaStart), B.curve.togaExp);
            const dx = Math.max(0, 1 - Math.abs((t - MID_CENTER) / (MID_WIDTH/2)));
            const bell = 1 + (MID_GAIN - 1) * Math.pow(dx, B.curve.bellExp);
            const idleBoosted = clamp(idleRaw * (1 + (bell - 1) * B.curve.idleBell));
            const n1Boosted   = clamp(n1Raw * (B.curve.n1BellMode === "bell" ? bell : 1) * B.curve.n1BellMul);
            const togaBoosted = Math.pow(togaRaw, B.curve.togaPow);
            const idleFinalRaw = Math.max(idleBoosted, MIN_LAYER_FLOOR * B.curve.idleFloorMul) * B.curve.idleVolumeBoost;
            const idleFinal    = Math.min(B.curve.maxLayerGain, idleFinalRaw * B.curve.idleFinalMul);
            const n1Final   = Math.max(n1Boosted * B.curve.n1Scale, MIN_LAYER_FLOOR * B.curve.n1FloorMul);
            const TOGA_BOOST = B.curve.togaBoost;  // try 1.4–2.0
            const togaFinal = Math.min(1, togaBoosted * TOGA_BOOST);
            const idleRate = 1 + (B.curve.idlePitchIntensity * Math.sqrt(Math.max(0, t * B.curve.idleRateMul))) * B.curve.idleRateTail;
            const n1Rate   = 1 + B.curve.n1RateK * t;
            const togaRate = 1 + B.curve.togaRateK * Math.pow(t, B.curve.togaRateExp);
            return {
              gains: { idle: idleFinal, n1: n1Final, toga: Math.min(1, togaFinal) },
              rates: { idle: idleRate, n1: n1Rate, toga: togaRate }
            };
          }
          function setGainSmooth(param, target){
            const now = geCtx.currentTime;
            param.cancelScheduledValues(now);
            param.setTargetAtTime(target, now, 0.04);
          }
// -------------------------------------------------------------
// BUZZSAW ALTITUDE ATTENUATION (per-bank curve) — smooth altitude → buzzsaw multiplier curve
// -------------------------------------------------------------
if (B.toga.altVolume) {
  // Altitude-based TOGA volume fade: linear multiplier from 1.0 at the
  // ground down to floor by maxFt. No filters involved.
  window['applyTogaAltitudeVolume_'+SFX] = function (alt) {
    const f = Math.max(0, Math.min(1, (alt || 0) / B.toga.altVolume.maxFt));
    return 1 - f * (1 - B.toga.altVolume.floor);
  };
}
if (B.toga.altitudeHF) {
  // Carried over verbatim from the A330/A339/A380 modules. Note: in the
  // original script this global is defined but never called, and no layer
  // ever receives a .togaHF node — it is inert. Kept for exact parity.
  window['applyTogaAltitudeHF_'+SFX] = function (layer, geCtx) {
    const alt = (window.geofs?.animation?.values?.altitude) || 0;
    const maxAlt = 8000;
    const f = 1 - (alt / maxAlt);
    const altFactor = Math.max(0, Math.min(1, f));
    const minHz = 250, maxHz = 20000;
    const hz = minHz + (maxHz - minHz) * Math.pow(altFactor, 1.4);
    if (layer.togaHF) layer.togaHF.frequency.setTargetAtTime(hz, geCtx.currentTime, 0.15);
  };
}
window['buzzsawAltitudeFactor_'+SFX] = function (alt) {
    const maxAlt = B.buzzsaw.altMaxFt;
    const f = 1 - (alt / maxAlt);
    return Math.max(0, Math.min(1, f));
};
window['applyBuzzsawAltitudeAttenuation_'+SFX] = function (buzz, amt, geCtx) {
    const alt = geofs.animation.values.altitude || 0;
    const altFactor = window['buzzsawAltitudeFactor_'+SFX](alt);
    // Smooth, less sensitive fade
    const buzzGain = amt * Math.pow(altFactor, B.buzzsaw.altExp) * B.buzzsaw.altScale;
    buzz.gainNode.gain.setTargetAtTime(buzzGain, geCtx.currentTime, 0.08);
};
          // -------------------------------------------------------
          // applyRawThrottle — includes wing2 attenuation
          // -------------------------------------------------------
          window[NP+'_applyRawThrottle'] = function(raw){
    const t = Math.max(0, Math.min(1, Number(raw) || 0));
    const targets = computeTargets(t);
 const view = window[IP+'_viewType'] || "exterior";
    let wing2Factor = 1.0;
    if      (view === "wing2")      wing2Factor = B.mix.wing2EngineAtten;
    else if (view === "wingEngine") wing2Factor = B.mix.wingEngineAtten;
    ['idle','n1','toga','buzzsaw'].forEach(k=>{
        const layer = window[NP+'_layers']?.[k];
        if (!layer) return;
// -----------------------------
// BUZZSAW LOGIC
// -----------------------------
if (k === "buzzsaw") {
    const buzz = window[NP+'_layers']?.buzzsaw;
    if (!buzz) return;
    const start = B.buzzsaw.throttleStart;
    const end   = B.buzzsaw.throttleEnd;
    let amt = 0;
    if (t > start) {
        amt = (t - start) / (end - start);
        amt = Math.pow(Math.min(1, Math.max(0, amt)), 1.4);
    }
    // Pitch is driven by throttle only — captured before the view-based
    // volume scaling below so changing the boost never bends the pitch.
    const pitchAmt = amt;
    // wingEngine view: right next to the engine, so 50% more buzzsaw
    // comes through than the (unchanged) wing2 view.
    if      (view === "wingEngine") amt *= B.buzzsaw.wingEngineBoost;
    else if (view === "wing2")      amt *= B.buzzsaw.wing2Atten;
    else if (view === "cockpit")    amt *= B.buzzsaw.cockpitAtten;
    // Apply altitude-based gain
    window['applyBuzzsawAltitudeAttenuation_'+SFX](buzz, amt * B.mix.engineLayerAtten, geCtx);
    // Pitch (multiplied by proximity pitch shift)
    const proximityPitch = window[IP+'_proximityPitchMult'] || 1.0;
    const rate = (B.buzzsaw.rateBase + pitchAmt * B.buzzsaw.rateSpan) * proximityPitch;
    if (buzz.src?.playbackRate) {
        buzz.src.playbackRate.setTargetAtTime(rate, geCtx.currentTime, 0.05);
    }
    return;
}
    // -----------------------------
    // NORMAL ENGINE LAYERS
    // -----------------------------
    let g = targets.gains[k] * (window[IP+'_master_boost'] || B.mix.masterBoost);
    if (k === "toga") {
      g *= B.mix.togaGain;
      if (B.toga.altVolume) {
        const _alt = (window.geofs?.animation?.values?.altitude) || 0;
        g *= window['applyTogaAltitudeVolume_'+SFX](_alt);
      }
      if (view === "cockpit") g *= B.toga.cockpitMult;
    }
    if (k === "idle") g *= (B.mix.idleGain || 1);
    if (k === "n1")   g *= (B.mix.n1Gain || 1);
    if (k === "idle" && view === "wing2") g *= B.mix.idleWing2Boost;
    g *= wing2Factor;
    g *= B.mix.engineLayerAtten;
    g = Math.max(0, Math.min(1.3, g));
    setGainSmooth(layer.gainNode.gain, g);
if (layer.src?.playbackRate) {
    const proximityPitch = window[IP+'_proximityPitchMult'] || 1.0;
    layer.src.playbackRate.setTargetAtTime(targets.rates[k] * proximityPitch, geCtx.currentTime, 0.04);
}
});
          };
          // -------------------------
          // RPM bind — hysteresis
          // -------------------------
          window[NP+'_enableRpmBind'] = function(){
            try {
              const engine = window.geofs?.aircraft?.instance?.engine;
              if (!engine) return;
              // APU interlock is installed once, globally, on
              // geofs.aircraft.Aircraft.prototype.startEngines — see
              // _GE_installApuStartEnginesGuard() near the top of the file.
              // Nothing to wire up here per-pack anymore.
              if (window['_ge90_hardwire_poll_'+SFX])
                clearInterval(window['_ge90_hardwire_poll_'+SFX]);
     const initialRpm = Number(engine.rpm) || 0;
              window[IP+'_lastRpm'] = initialRpm;
              window[IP+'_lastEngineOn'] = !!engine.on;
              window[IP+'_engineOff'] = !engine.on || initialRpm <= B.engine.rpmShutdown;
              window[IP+'_engineOffPlayed'] = window[IP+'_engineOff'];
              window[IP+'_spawnTime'] = performance.now();
            // Reset arm state, stall counter, and suppress window on every spawn
             _engineStateArmed = false;
              _engineStateArmTicks = 0;
   window[IP+'_stallTicks'] = 0;
              window[IP+'_lastValidRpm'] = 0;
              window[IP+'_rpmHistory'] = [];
              window[IP+'_lastEngineOn'] = !!engine.on;
             window[IP+'_startupPlaying'] = false;
              window[IP+'_startupKilled'] = false;
              startTime = performance.now();
              window['_ge90_hardwire_poll_'+SFX] = setInterval(()=>{
                try {
               const raw = Number(engine.rpm);
                  if (!isFinite(raw)) return;
                 // Discard transitions during the arming window (swallows spawn false-triggers)
                  if (!_engineStateArmed) {
                      _engineStateArmTicks++;
                      if (_engineStateArmTicks >= ENGINE_ARM_TICKS) {
                          _engineStateArmed = true;
                      } else {
                          window[IP+'_lastRpm'] = Math.max(0, raw);
                          window[IP+'_engineOff'] = window[IP+'_lastRpm'] <= B.engine.rpmShutdown;
                          return;
                      }
                  }
// Terrain-load stall detection using rolling RPM history
                  // Distinguishes terrain snap-to-zero from genuine shutdown
                  if (!window[IP+'_stallTicks']) window[IP+'_stallTicks'] = 0;
                  if (!window[IP+'_rpmHistory']) window[IP+'_rpmHistory'] = [];
                  // Keep a rolling history of the last 4 RPM readings
                  window[IP+'_rpmHistory'].push(raw);
                  if (window[IP+'_rpmHistory'].length > 4) {
                      window[IP+'_rpmHistory'].shift();
                  }
                  if (raw > B.engine.rpmShutdown) {
                      // RPM healthy - reset stall counter
                      window[IP+'_stallTicks'] = 0;
                  } else if (!window[IP+'_engineOff']) {
                      window[IP+'_stallTicks']++;
                      const history = window[IP+'_rpmHistory'];
                      const prevRpm = history.length >= 2
                          ? history[history.length - 2]
                          : (window[IP+'_lastRpm'] || 0);
                      const wasHealthy = prevRpm > B.engine.rpmShutdown;
                      // Terrain stall signature: RPM snaps specifically to near-zero
                      // (not just below threshold - a real idle floor won't be near zero)
                      const isNearZero = raw <= 50;
                      const isSnapDrop = wasHealthy && isNearZero;
                      if (isSnapDrop && window[IP+'_stallTicks'] <= 4) {
                          window[IP+'_lastRpm'] = 0;
                          return;
                      }
                      // Not a terrain snap - fall through to shutdown trigger
                  }
                  const rpm = Math.max(0, raw);
                  const wasOff = window[IP+'_engineOff'];
                  const timeSinceSpawn = performance.now() - window[IP+'_spawnTime'];
                  const spawnProtectionActive = timeSinceSpawn < B.engine.spawnProtectionMs;
                  // Check altitude to prevent shutdown sound when spawning in air
                  let altitudeProtectionActive = false;
                  try {
                    const altitude = window.geofs?.aircraft?.instance?.altitude || 0;
                    altitudeProtectionActive = altitude > B.engine.altitudeProtectionFt;
                  } catch(e) {
                    // If altitude check fails, assume no protection
                  }
               const engineOnNow = !!engine.on;
                  const nowOff = !engineOnNow;
                  const nowOn  = engineOnNow;
                  // APU interlock backstop: _GE_installApuStartEnginesGuard()
                  // patching startEngines() is the primary defense, so this is
                  // defense-in-depth for anything that sets engine.on directly.
                  // Reverts the flip before the startup trigger below.
                  if (wasOff && nowOn && window[IP+'_lastEngineOn'] === false &&
                      window[IP+'_lastRpm'] < B.engine.rpmMin && !_GE_apuAllowsEngineStart()) {
                    const _apuInst = window.geofs?.aircraft?.instance;
                    try { if (typeof _apuInst?.stopEngine === 'function') _apuInst.stopEngine(); } catch(e){}
                    try { engine.on = false; } catch(e){}
                    _GE_apuInterlockNotice();
                    window[IP+'_engineOff'] = true;
                    window[IP+'_lastRpm'] = rpm;
                    window[IP+'_lastEngineOn'] = false;
                    return;
                  }
                  if (window[IP+'_startupSrc'] && rpm >= B.engine.rpmMin) {
                    try { window[IP+'_startupSrc'].stop(); } catch(e){}
                    window[IP+'_startupSrc'] = null;
                  }
                  // Startup trigger: engine.on just flipped true AND last rpm reading
                  // was below B.engine.rpmMin (engine was genuinely off/spooling, not already running)
                if (wasOff && nowOn && window[IP+'_lastRpm'] < B.engine.rpmMin) {
                    // Suppress engine layers during startup sequence
                    window[IP+'_startupPlaying'] = true;
                    playStartupDual();
                    window[IP+'_engineOffPlayed'] = false;
                    window[IP+'_engineOff'] = false;
                    window[IP+'_lastRpm'] = rpm;
                    ['idle','n1','toga','buzzsaw'].forEach(k=>{
                      const layer = window[NP+'_layers']?.[k];
                      if (!layer) return;
                      try {
                        layer.gainNode.gain.cancelScheduledValues(geCtx.currentTime);
                        layer.gainNode.gain.setValueAtTime(0, geCtx.currentTime);
                        layer.gainNode.gain.setTargetAtTime(0.001, geCtx.currentTime, 0.25);
                      } catch(e){}
                    });
                  }
                else if (!wasOff &&
                           nowOff &&
                           window[IP+'_lastEngineOn'] === true &&
                           !spawnProtectionActive &&
                           !altitudeProtectionActive) {
                    if (!window[IP+'_engineOffPlayed']) {
                      const altitude = window.geofs?.aircraft?.instance?.altitude || 0;
                      // Quieter shutdown
                      playOneShot(B.assets.shutdown, 0.20);
                      window[IP+'_engineOffPlayed'] = true;
                    }
                    ['idle','n1','toga','buzzsaw'].forEach(k=>{
                      const layer = window[NP+'_layers']?.[k];
                      if (layer)
                        layer.gainNode.gain.setTargetAtTime(0, geCtx.currentTime, 0.25);
                    });
                    window[IP+'_engineOff'] = true;
                    window[IP+'_lastRpm'] = rpm;
                    if (window[IP+'_startupSrc']) {
                      try { window[IP+'_startupSrc'].stop(); } catch(e){}
                      window[IP+'_startupSrc'] = null;
                    }
                    return;
                  }
                  window[IP+'_engineOff'] = nowOff;
                  window[IP+'_lastRpm'] = rpm;
                  window[IP+'_lastEngineOn'] = engineOnNow;
                  if (nowOff) {
                    ['idle','n1','toga','buzzsaw'].forEach(k=>{
                      const layer = window[NP+'_layers']?.[k];
                      if (layer)
                        layer.gainNode.gain.setTargetAtTime(0, geCtx.currentTime, 0.25);
                    });
                 } else {
    if (window[IP+'_startupIntSrc'] && rpm >= B.engine.rpmMin) {
        try { window[IP+'_startupIntSrc'].stop(); } catch(e){}
        window[IP+'_startupIntSrc'] = null;
    }
    // Suppress engine layers while startup sequence is playing
    if (window[IP+'_startupPlaying']) {
        ['idle','n1','toga','buzzsaw'].forEach(k => {
            const layer = window[NP+'_layers']?.[k];
            if (layer) layer.gainNode.gain.setTargetAtTime(0, geCtx.currentTime, 0.1);
        });
        return;
    }
// ------------------------------------------------------
// Kill ALL startup sounds once engine layers take over
// ------------------------------------------------------
if (!window[IP+'_startupKilled']) {
    try {
        if (window[IP+'_startupSrc']) {
            try { window[IP+'_startupSrc'].stop(); } catch(e){}
            window[IP+'_startupSrc'] = null;
        }
        if (window[IP+'_startupIntSrc']) {
            try { window[IP+'_startupIntSrc'].stop(); } catch(e){}
            window[IP+'_startupIntSrc'] = null;
        }
    } catch(e){}
    window[IP+'_startupKilled'] = true;
}
                const norm = Math.max(0, Math.min(1, (rpm - B.engine.rpmMin) / (9200 - B.engine.rpmMin)));
                    window[NP+'_applyRawThrottle'](norm);
                  }
                } catch(e){}
              }, 60);
            } catch(e){
              console.warn('[GE90] enableRpmBind failed', e);
            }
          };
          // -------------------------
          // Initialize layers then bind RPM
          // -------------------------
          Promise.all([
            createLayer('idle'),
            createLayer('n1'),
            createLayer('toga'),
            createBuzzsawLayer()
          ]).then(()=>{
            window.dispatchEvent(new CustomEvent('GE90:ready'));
            setTimeout(()=>window[NP+'_enableRpmBind'](), 1200);
          }).catch(e => {
            console.warn('[GE90] start failed', e);
          });
        } catch(e){
          console.warn('[GE90] start failed', e);
        }
      };
// Shared cross-pack coordination: native GeoFS audio stays muted while ANY pack claims the aircraft, restored only when none do.
      window._GEPacks = window._GEPacks || {};
      window._GEPacks[CODE] = false;
      function muteNativeAudio(){
        try { window.audio?.mute?.(); } catch(e){}
      }
      function unmuteNativeAudio(){
        try { window.audio?.unmute?.(); } catch(e){}
      }
      if (!window._GEArbiterInstalled) {
        window._GEArbiterInstalled = true;
        window._GEArbiterLastActive = null;
        setInterval(() => {
          try {
            const anyActive = Object.values(window._GEPacks).some(v => v === true);
            if (anyActive) {
              // A pack is replacing native engine sound — keep GeoFS's
              // own audio suppressed for as long as that's true.
              window.audio?.mute?.();
              window._GEArbiterLastActive = true;
            } else if (window._GEArbiterLastActive !== false) {
              // Just went inactive: restore native audio once. Don't keep forcing unmute — GeoFS's own 's' key also controls it.
              window.audio?.unmute?.();
              window._GEArbiterLastActive = false;
            }
          } catch(e){}
        }, 150);
      }
      // -------------------------
      // Aircraft-gated startup
      // -------------------------
      function watchAircraftId(){
        let lastId = undefined;
        const evaluate = () => {
          try {
            const inst = window.geofs?.aircraft?.instance;
            const idRaw = inst?.id;
            if (idRaw === undefined || idRaw === null) return;
            const id = String(idRaw);
            if (id === lastId) return;
            lastId = id;
            const matches = B.ids.includes(id);
            if (matches) {
              window[IP+'_active'] = true;
              window._GEPacks[CODE] = true;
              muteNativeAudio();
              if (!window[IP+'_started']) {
                window[IP+'_started'] = true;
                if (document.readyState === 'loading') {
                  document.addEventListener('DOMContentLoaded', start);
                } else {
                  start();
                }
              } else {
                if (!window[IP+'_userPaused']) { try { window[IP+'_audioCtx']?.resume(); } catch(e){} }
              }
            } else {
              window[IP+'_active'] = false;
              window._GEPacks[CODE] = false;
              unmuteNativeAudio();
              try {
                const geCtx = window[IP+'_audioCtx'];
                const m = window[IP+'_master'];
                if (geCtx && m) {
                  m.gain.cancelScheduledValues(geCtx.currentTime);
                  m.gain.setValueAtTime(0, geCtx.currentTime);
                }
              } catch(e){}
              try { window[IP+'_audioCtx']?.suspend(); } catch(e){}
            }
          } catch(e){}
        };
        let ticks = 0;
        const fastPoll = setInterval(() => {
          evaluate();
          ticks++;
          if (ticks > 100) {
            clearInterval(fastPoll);
            setInterval(evaluate, 200);
          }
        }, 100);
      }
      watchAircraftId();
    } catch(e){
      console.warn('[GE90] initGE90 failed', e);
    }
  })();
  // -------------------------
  // Page-context logic: interior/exterior processing, ambience, realism, S/P, gear
  // -------------------------
  (function setupPageContext(){
    function ready(cb){
      if (window[NP+'_layers'] && window[IP+'_audioCtx'] && window[IP+'_master']) return cb();
      const onReady = () => { window.removeEventListener('GE90:ready', onReady); cb(); };
      window.addEventListener('GE90:ready', onReady);
      const poll = setInterval(()=>{
        if (window[NP+'_layers'] && window[IP+'_audioCtx'] && window[IP+'_master']){
          clearInterval(poll);
          window.removeEventListener('GE90:ready', onReady);
          cb();
        }
      }, 200);
    }
    ready(() => {
      try {
        const ctx = window[IP+'_audioCtx'];
        const master = window[IP+'_master'];
        // -------------------------
        // View detection
        // -------------------------
   function getViewType(){
  try {
    const cam = window.geofs?.camera;
    if (!cam) return "exterior";
    let s = cam.currentModeName || cam.currentView || cam.currentDefinition?.name || cam.mode || "";
    s = String(s).toLowerCase();
    if (!s) return "exterior";
    const V = B.views;
    if (V.cockpit.some(k => s.includes(k))) return "cockpit";
    if (V.hasWingEngine &&
        V.side.some(k => s.includes(k)) &&
        V.engine.some(k => s.includes(k))) return "wingEngine";
    if (V.wing2.some(k => s.includes(k))) return "wing2";
    return "exterior";
  } catch(e){
    return "exterior";
  }
}
        // -------------------------
        // Loudness chain (pre + compressor)
        // -------------------------
        if (!window[IP+'_loudness_nodes']){
          try {
            const PRE_GAIN = Math.pow(10, B.mix.preGainDb/20);
            const pre = ctx.createGain();
            pre.gain.setValueAtTime(PRE_GAIN, ctx.currentTime);
            pre._GE90_TAG = true;
            const comp = ctx.createDynamicsCompressor();
            comp.threshold.setValueAtTime(B.mix.compThresholdDb, ctx.currentTime);
            comp.ratio.setValueAtTime(B.mix.compRatio, ctx.currentTime);
            comp.attack.setValueAtTime(0.008, ctx.currentTime);
            comp.release.setValueAtTime(0.30, ctx.currentTime);
            comp._GE90_TAG = true;
            master.disconnect();
            window[IP+'_loudness_nodes'] = { pre, comp };
          } catch(e){
            console.warn('[GE90] loudness node insert failed', e);
          }
        }
        const { pre, comp } = window[IP+'_loudness_nodes'];
        // -------------------------
        // INTERIOR AUDIO CHAIN + REALISM + SMOOTHING
        // -------------------------
        if (!window[IP+'_interior_nodes']){
          try {
            const extGain = ctx.createGain();
            extGain.gain.setValueAtTime(1, ctx.currentTime);
            extGain._GE90_TAG = true;
            const intGain = ctx.createGain();
            intGain.gain.setValueAtTime(0, ctx.currentTime);
            intGain._GE90_TAG = true;
            const intLPF = ctx.createBiquadFilter();
            intLPF.type = "lowpass";
            intLPF.frequency.setValueAtTime(4000, ctx.currentTime);
            intLPF._GE90_TAG = true;
            const intEQ = ctx.createBiquadFilter();
            intEQ.type = "peaking";
            intEQ.frequency.setValueAtTime(450, ctx.currentTime);
            intEQ.gain.setValueAtTime(3.0, ctx.currentTime);
            intEQ.Q.setValueAtTime(1.5, ctx.currentTime);
            intEQ._GE90_TAG = true;
            const intLowShelf = ctx.createBiquadFilter();
            intLowShelf.type = "lowshelf";
            intLowShelf.frequency.setValueAtTime(180, ctx.currentTime);
            intLowShelf.gain.setValueAtTime(5.0, ctx.currentTime);
            intLowShelf._GE90_TAG = true;
            const intHighShelf = ctx.createBiquadFilter();
            intHighShelf.type = "highshelf";
            intHighShelf.frequency.setValueAtTime(2500, ctx.currentTime);
            intHighShelf.gain.setValueAtTime(-0.5, ctx.currentTime);
            intHighShelf._GE90_TAG = true;
            const intComp = ctx.createDynamicsCompressor();
              intComp.threshold.setValueAtTime(B.mix.compThresholdDb, ctx.currentTime); // -12 dB
              intComp.ratio.setValueAtTime(B.mix.compRatio, ctx.currentTime);            // 4:1
              intComp.attack.setValueAtTime(0.003, ctx.currentTime);                // faster attack
              intComp.release.setValueAtTime(0.12, ctx.currentTime);
            intComp._GE90_TAG = true;
            const intSmooth = ctx.createGain();
            intSmooth.gain.setValueAtTime(1, ctx.currentTime);
            intSmooth._GE90_TAG = true;
            const intVibe = ctx.createGain();
            intVibe.gain.setValueAtTime(1, ctx.currentTime);
            intVibe._GE90_TAG = true;
            const lfo = ctx.createOscillator();
            const lfoGain = ctx.createGain();
            lfo.frequency.setValueAtTime(B.mix.wing2VibeRate, ctx.currentTime);
            lfoGain.gain.setValueAtTime(B.mix.wing2VibeDepth, ctx.currentTime);
            lfo.connect(lfoGain);
            lfoGain.connect(intVibe.gain);
            lfo.start();
            master.connect(extGain);
            master.connect(intGain);
            intGain.connect(intLPF);
            intLPF.connect(intEQ);
            intEQ.connect(intLowShelf);
            intLowShelf.connect(intHighShelf);
            intHighShelf.connect(intComp);
            intComp.connect(intVibe);
            intVibe.connect(intSmooth);
            intSmooth.connect(pre);
            extGain.connect(pre);
            // --- FINAL MASTER GAIN (for smooth global volume control) ---
window[IP+'_finalMaster'] = ctx.createGain();
window[IP+'_finalMaster'].gain.value = 1.0;
// Rewire chain
pre.connect(comp);
comp.connect(window[IP+'_finalMaster']);
// True peak limiter — catches any signal that exceeds 0dBFS before
// it reaches the output stage, preventing digital clipping at high slider values
const _packLimiter = ctx.createDynamicsCompressor();
_packLimiter.threshold.value = -1.0;  // start limiting just below 0dBFS
_packLimiter.knee.value      =  0.0;  // hard knee — instant limiting, no soft curve
_packLimiter.ratio.value     = 20.0;  // 20:1 ratio = effectively a brick wall limiter
_packLimiter.attack.value    = 0.001; // 1ms attack — catches peaks almost instantly
_packLimiter.release.value   = 0.1;   // 100ms release — fast enough to not pump
_packLimiter._GE90_TAG       = true;
window[IP+'_limiter']          = _packLimiter;
window[IP+'_finalMaster'].connect(_packLimiter);
_packLimiter.connect(ctx.destination);
            window[IP+'_interior_nodes'] = {
              extGain, intGain, intLPF, intEQ, intLowShelf, intHighShelf,
              intComp, intSmooth, intVibe, lfo, lfoGain
            };
// ------------------------------------------------------
// Gear toggle click sound (cockpit + jumpseat only)
// ------------------------------------------------------
let _lastGearTarget = null;
setInterval(() => {
    try {
        const target = window.geofs?.animation?.values?.gearTarget;
        if (target == null) return;
        if (_lastGearTarget === null) {
            _lastGearTarget = target;
            return;
        }
        // Detect gear command change (0 -> 1 or 1 -> 0)
        if (target !== _lastGearTarget) {
            _lastGearTarget = target;
            // Check view
            const view = (typeof getViewType === "function")
                ? getViewType()
                : (window[IP+'_viewType'] || "exterior");
if (view === "cockpit" || view === "jumpseat" || view === "wing2") {
    if (window[NP+'_playGearClick']) {
        window[NP+'_playGearClick']();
    }
}
        }
    } catch(e){}
}, 120);
          } catch(e){
            console.warn("[GE90] interior chain failed", e);
          }
        }
        // Gear movement hum (cockpit/wing2/wingEngine): continuous fade-in/out loop through this pack's own master gain node.
        let _gearHumBuf     = null;
        let _gearHumLoading = false;
        let _gearHumSrc     = null;
        let _gearHumGain    = null;
        let _gearHumActive  = false;
        function _makeSeamlessGearLoop(srcBuffer, fadeSec){
          try {
            const sampleRate  = srcBuffer.sampleRate;
            const fadeSamples = Math.floor(fadeSec * sampleRate);
            const length      = srcBuffer.length;
            if (fadeSamples * 2 >= length) return srcBuffer;
            const out = ctx.createBuffer(srcBuffer.numberOfChannels, length, sampleRate);
            for (let ch = 0; ch < srcBuffer.numberOfChannels; ch++){
              const inData  = srcBuffer.getChannelData(ch);
              const outData = out.getChannelData(ch);
              for (let i = 0; i < length; i++) outData[i] = inData[i];
              for (let i = 0; i < fadeSamples; i++){
                const fadeOut = 1 - (i / fadeSamples);
                const fadeIn  = i / fadeSamples;
                const tailIdx = length - fadeSamples + i;
                outData[tailIdx] = inData[tailIdx] * fadeOut + inData[i] * fadeIn;
              }
            }
            return out;
          } catch(e){ return srcBuffer; }
        }
        async function _loadGearHum(){
          if (_gearHumBuf || _gearHumLoading) return;
          _gearHumLoading = true;
          try {
            const r = await fetch(B.assets.gearHum, { mode: "cors" });
            if (!r.ok) throw new Error(`HTTP ${r.status}`);
            const ab = await r.arrayBuffer();
            const decoded = await ctx.decodeAudioData(ab);
            _gearHumBuf = _makeSeamlessGearLoop(decoded, 0.02);
          } catch(e){
            console.warn("[GE90 Gear] Hum buffer failed to load", e);
          } finally {
            _gearHumLoading = false;
          }
        }
        _loadGearHum();
        function _gearVolumeForView(view){
          if (view === "cockpit")    return B.gear.volCockpit;
          if (view === "wing2")      return B.gear.volWing2;
          if (view === "wingEngine") return B.gear.volWingEngine;
          return 0;
        }
        async function playGearMovement(view){
          try {
            if (!_gearHumBuf) { await _loadGearHum(); if (!_gearHumBuf) return; }
            const targetVol = _gearVolumeForView(view);
            if (targetVol <= 0) { stopGearSoundSmooth(); return; }
            if (_gearHumActive && _gearHumGain) {
              // Already running — just retarget volume (e.g. view changed
              // from cockpit to wing2 mid-retraction).
              _gearHumGain.gain.cancelScheduledValues(ctx.currentTime);
              _gearHumGain.gain.setTargetAtTime(targetVol, ctx.currentTime, 0.15);
              return;
            }
            const src = ctx.createBufferSource();
            src.buffer = _gearHumBuf;
            src.loop = true;
            src._GE90_TAG = true;
            const lpf = ctx.createBiquadFilter();
            lpf.type = "lowpass";
            lpf.frequency.value = B.gear.lpfFreq;
            lpf.Q.value = B.gear.lpfQ;
            lpf._GE90_TAG = true;
            const g = ctx.createGain();
            g.gain.setValueAtTime(0, ctx.currentTime);
            g._GE90_TAG = true;
            src.connect(lpf);
            lpf.connect(g);
            g.connect(master);
            src.start(0);
            g.gain.linearRampToValueAtTime(targetVol, ctx.currentTime + B.gear.fadeIn);
            _gearHumSrc    = src;
            _gearHumGain   = g;
            _gearHumActive = true;
          } catch(e){
            console.warn("[GE90 Gear] playGearMovement failed", e);
          }
        }
        function stopGearSoundSmooth(){
          if (!_gearHumActive) return;
          try {
            const src = _gearHumSrc;
            const g   = _gearHumGain;
            _gearHumActive = false;
            if (g) {
              g.gain.cancelScheduledValues(ctx.currentTime);
              g.gain.setValueAtTime(g.gain.value, ctx.currentTime);
              g.gain.linearRampToValueAtTime(0, ctx.currentTime + B.gear.fadeOut);
            }
            setTimeout(() => {
              try { if (src) { src.stop(0); src.disconnect(); } } catch(e){}
              try { if (g) g.disconnect(); } catch(e){}
            }, (B.gear.fadeOut * 1000) + 60);
            _gearHumSrc  = null;
            _gearHumGain = null;
          } catch(e){
            console.warn("[GE90 Gear] stopGearSoundSmooth failed", e);
          }
        }
        // Continuous gear-position poll: starts/stops the hum while the
        // gear is actually retracting/extending (not just on command),
        // in whichever view is currently audible.
        setInterval(() => {
          try {
            const gearPos = window.geofs?.animation?.values?.gearPosition;
            if (typeof gearPos !== "number") return;
            const view    = getViewType();
            const moving  = gearPos > 0.01 && gearPos < 0.99;
            const audible = (view === "cockpit" || view === "wing2" || view === "wingEngine");
            if (moving && audible) {
              playGearMovement(view);
            } else {
              stopGearSoundSmooth();
            }
          } catch(e){}
        }, 120);
        // Tire screech on touchdown (main gear via groundContact, nose gear via pitch-settle heuristic); cockpit/jumpseat/wing views only, 45% volume.
        const TIRE_SCREECH_NOSE_ATILT_BAND = 0.5; // degrees from level treated as "nose wheel down"
        let _tireScreechBuf     = null;
        let _tireScreechLoading = false;
        async function _loadTireScreech(){
          if (_tireScreechBuf || _tireScreechLoading) return;
          _tireScreechLoading = true;
          try {
            const r = await fetch(B.assets.tireScreech, { mode: "cors" });
            if (!r.ok) throw new Error(`HTTP ${r.status}`);
            const ab = await r.arrayBuffer();
            _tireScreechBuf = await ctx.decodeAudioData(ab);
          } catch(e){
            console.warn("[GE90 Tire Screech] Buffer failed to load", e);
          } finally {
            _tireScreechLoading = false;
          }
        }
        _loadTireScreech();
        function _tireScreechVolumeForView(view){
          if (view === "cockpit" || view === "jumpseat" || view === "wing2" || view === "wingEngine") {
            return B.mix.tireScreechVol;
          }
          return 0;
        }
        async function _playTireScreech(label){
          try {
            if (!_tireScreechBuf) { await _loadTireScreech(); if (!_tireScreechBuf) return; }
            const view = (typeof getViewType === "function")
                ? getViewType()
                : (window[IP+'_viewType'] || "exterior");
            const vol = _tireScreechVolumeForView(view);
            if (vol <= 0) return;
            const src = ctx.createBufferSource();
            src.buffer = _tireScreechBuf;
            src._GE90_TAG = true;
            const g = ctx.createGain();
            g.gain.setValueAtTime(vol, ctx.currentTime);
            g._GE90_TAG = true;
            src.connect(g);
            g.connect(master);
            src.start(0);
            src.onended = () => { try { src.disconnect(); } catch(e){} try { g.disconnect(); } catch(e){} };
          } catch(e){
            console.warn("[GE90 Tire Screech] playback failed", e);
          }
        }
        let _tsWasGrounded  = false;
        let _tsAwaitingNose = false;
        setInterval(() => {
          try {
            const inst = window.geofs?.aircraft?.instance;
            if (!inst) return;
            const grounded = !!inst.groundContact;
            const atilt = window.geofs?.animation?.values?.atilt;
            if (grounded && !_tsWasGrounded) {
              _playTireScreech("main gear touchdown");
              _tsAwaitingNose = true;
            }
            if (!grounded) {
              _tsAwaitingNose = false;
            }
            if (_tsAwaitingNose && grounded && typeof atilt === "number" && Math.abs(atilt) <= TIRE_SCREECH_NOSE_ATILT_BAND) {
              _playTireScreech("nose gear touchdown");
              _tsAwaitingNose = false;
            }
            _tsWasGrounded = grounded;
          } catch(e){}
        }, 100);
        const {
          extGain, intGain, intLPF, intLowShelf, intHighShelf,
          intSmooth, intVibe, lfoGain
        } = window[IP+'_interior_nodes'];
// -----------------------------
// Runway rattle/creak system (groundContact-driven)
// -----------------------------
(function(){
  const URLS = B.assets.rattle;
  // Polling / tuning
  const POLL_MS = 100;
  const SPEED_START_KTS = 25;
  const SPEED_MIN_KTS = 30;
  const SPEED_STOP_KTS = 22;
  const ACCEL_START = 0.20;
  const ACCEL_STOP  = 0.08;
  const ACCEL_HOLD_MS = 120;
  const SPAWN_PROTECT_MS = 3000;
  const LOOP_FADE_LONG = 0.8;
  const LOOP_FADE = 0.28;
  const MAX_RATTLE_BUS_GAIN = 0.9
  const TRANSIENT_INTERVAL_MIN_MS = 300;
  const TRANSIENT_INTERVAL_MAX_MS = 900;
  const TRANSIENT_COOLDOWN_MS = 700;
  const _buf = {};
  let _poll = null;
  let _lastSpeedKts = null;
  let _lastSpeedTs = null;
  let _accelStartTime = 0;
  let _rLoop = null;
  let _transTimer = null;
  const _transCooldown = { transC: 0 };
  // Exposed state
  window[IP+'_rattle_state'] = window[IP+'_rattle_state'] || {};
  window[IP+'_rattle_state'].active = false;
  window[IP+'_rattle_bufs'] = window[IP+'_rattle_bufs'] || {};
  function _getOutNode(){
    return (window[IP+'_interior_nodes'] && window[IP+'_interior_nodes'].intGain) || window[IP+'_finalMaster'] || master || (ctx && ctx.destination);
  }
  // -------------------------
  // Helpers: reads from geofs instance robustly
  // -------------------------
  function _readInst(){ return window.geofs && window.geofs.aircraft && window.geofs.aircraft.instance ? window.geofs.aircraft.instance : null; }
  function _readGroundContact(){
    try {
      const inst = _readInst();
      if (!inst) return null;
      // direct property if present
      if (typeof inst.groundContact !== 'undefined') return !!inst.groundContact;
      // older/alternate names
      if (typeof inst.wheelsOnGround !== 'undefined') return !!inst.wheelsOnGround;
      if (typeof inst.wheelContact !== 'undefined') return !!inst.wheelContact;
      // fallback: undefined (caller will use conservative detection)
      return null;
    } catch(e){ return null; }
  }
  function _readGearPos(){
    try {
      if (typeof window.geofs?.animation?.values?.gearPosition === 'number') return window.geofs.animation.values.gearPosition;
      const inst = _readInst();
      if (inst && typeof inst.gearPosition === 'number') return inst.gearPosition;
    } catch(e){}
    return null;
  }
  function _readAltitude(){
    try {
      const inst = _readInst();
      if (!inst) return 0;
      return Number(inst.altitude || inst.alt || 0) || 0;
    } catch(e){ return 0; }
  }
  function _readGroundspeedKts(){
    try {
      const inst = _readInst();
      if (!inst) return null;
      const candidates = [
        inst.groundspeed, inst.groundSpeed, inst.speed, inst.airspeed,
        inst.groundspeedKts, inst.speedKts, inst.airspeedKts
      ];
      for (const c of candidates) if (typeof c === 'number' && !isNaN(c)) return Number(c);
      const vel = inst.velocity || inst.vel || null;
      if (vel && typeof vel.x === 'number') {
        const vx = vel.x||0, vy = vel.y||0, vz = vel.z||0;
        const mps = Math.sqrt(vx*vx + vy*vy + vz*vz);
        return mps / 0.514444;
      }
      return null;
    } catch(e){ return null; }
  }
  // Conservative fallback ground test (if groundContact is unavailable)
  function _isOnGroundFallback(){
    try {
      const inst = _readInst();
      if (!inst) return false;
      // wheel props
      const wheelProps = [inst.wheelsOnGround, inst.wheelsOn, inst.wheelContact, window.geofs?.animation?.values?.wheelContact, window.geofs?.animation?.values?.wheelsOnGround];
      for (const p of wheelProps) {
        if (typeof p === 'boolean') return p;
        if (typeof p === 'number') return p > 0;
      }
      const gearPos = _readGearPos();
      if (typeof gearPos === 'number' && gearPos > 0.6) return false;
      const alt = _readAltitude();
      const speed = _readGroundspeedKts() || 0;
      if (!isNaN(alt) && alt < 50 && speed < 40) return true;
    } catch(e){}
    return false;
  }
  // -------------------------
  // Audio loading / playback
  // -------------------------
  async function _loadAll(){
    const keys = Object.keys(URLS);
    await Promise.all(keys.map(async k=>{
      if (_buf[k]) return;
      try {
        const r = await fetch(URLS[k], { mode: 'cors' });
        if (!r.ok) { console.warn(`[GE90 Rattle] HTTP ${r.status} when fetching ${k}`); return; }
        const ab = await r.arrayBuffer();
        try {
          _buf[k] = await ctx.decodeAudioData(ab);
        } catch(decodeErr){
          console.warn(`[GE90 Rattle] decodeAudioData failed for ${k}`, decodeErr);
        }
      } catch(e){
        console.warn('[GE90 Rattle] fetch/load failed', k, e);
      }
    }));
    try { window[IP+'_rattle_bufs'] = Object.keys(_buf).reduce((o,k)=>{ o[k]=!!_buf[k]; return o; }, {}); } catch(e){}
  }
  function _playOneShot(buf, gain=0.22, pitchJitter=0.06, startOffset = 0, baseRate = 1.0, hpFreq = 120){
    try {
      if (!ctx || !buf) return;
      const src = ctx.createBufferSource();
      src.buffer = buf;
      src.loop = false;
      const pr = baseRate * (1 + (Math.random()*2-1) * pitchJitter);
      src.playbackRate.setValueAtTime(pr, ctx.currentTime);
      src._GE90_TAG = true;
      let nodeHead = src;
      let hp = null;
      if (hpFreq && hpFreq > 20) {
        hp = ctx.createBiquadFilter();
        hp.type = 'highpass';
        hp.frequency.setValueAtTime(hpFreq, ctx.currentTime);
        hp.Q.setValueAtTime(0.7, ctx.currentTime);
        hp._GE90_TAG = true;
        src.connect(hp);
        nodeHead = hp;
      }
      const g = ctx.createGain();
      g.gain.setValueAtTime(gain, ctx.currentTime);
      g._GE90_TAG = true;
      nodeHead.connect(g);
      g.connect(_getOutNode());
      const maxOffset = Math.max(0, Math.min((buf.duration||0)*0.08, startOffset || 0));
      src.start(ctx.currentTime, maxOffset);
      setTimeout(()=>{
        try { src.stop(); } catch(e){}
        try { src.disconnect(); } catch(e){}
        try { if (hp) hp.disconnect(); } catch(e){}
        try { g.disconnect(); } catch(e){}
      }, Math.round((buf.duration || 1.5)*1000 + 300));
    } catch(e){
      console.warn('[GE90 Rattle] one-shot play failed', e);
    }
  }
  // -------------------------
  // Loop control: start / update / stop
  // -------------------------
  function _startLoop(intensity){
    if (!ctx || _rLoop) return;
    if (!_buf.low || !_buf.midL) return;
    const lowSrc = ctx.createBufferSource();
    lowSrc.buffer = window._GE_seamlessLoopBuffer(ctx, _buf.low, 0.3);
    lowSrc.loop = true;
    lowSrc.playbackRate.setValueAtTime(1 + (Math.random()*0.02-0.01), ctx.currentTime);
    lowSrc._GE90_TAG = true;
    const lowG = ctx.createGain();
    lowG.gain.setValueAtTime(0, ctx.currentTime);
    lowG._GE90_TAG = true;
    lowSrc.connect(lowG);
    lowG.connect(_getOutNode());
    lowSrc.start(ctx.currentTime + Math.random()*0.12);
    const midSrc = ctx.createBufferSource();
    midSrc.buffer = window._GE_seamlessLoopBuffer(ctx, _buf.midL, 0.3);
    midSrc.loop = true;
    midSrc.playbackRate.setValueAtTime(1.22 + (Math.random()*0.04-0.02), ctx.currentTime);
    midSrc._GE90_TAG = true;
    const midHP = ctx.createBiquadFilter();
    midHP.type = 'highpass';
    midHP.frequency.setValueAtTime(140, ctx.currentTime);
    midHP.Q.setValueAtTime(0.7, ctx.currentTime);
    midHP._GE90_TAG = true;
    const midG = ctx.createGain();
    midG.gain.setValueAtTime(0, ctx.currentTime);
    midG._GE90_TAG = true;
    midSrc.connect(midHP);
    midHP.connect(midG);
    midG.connect(_getOutNode());
    midSrc.start(ctx.currentTime + Math.random()*0.12);
    _rLoop = { low:{src:lowSrc,gain:lowG}, mid:{src:midSrc,gain:midG, hp: midHP} };
    window[IP+'_rattle_state'].active = true;
    _updateLoopIntensity(intensity, LOOP_FADE_LONG);
    _scheduleNextTransient();
  }
  function _updateLoopIntensity(intensity, fadeSec){
    if (!_rLoop) return;
    intensity = Math.max(0, Math.min(1, intensity));
    fadeSec = (typeof fadeSec === 'number') ? fadeSec : LOOP_FADE;
    const lowTarget = 0.02 + intensity * 0.28;
    const midTarget = 0.06 + intensity * 0.50;
    const combined = Math.min(MAX_RATTLE_BUS_GAIN, lowTarget + midTarget);
    const scale = combined / (lowTarget + midTarget || 1);
    const now = ctx.currentTime;
    _rLoop.low.gain.gain.cancelScheduledValues(now);
    _rLoop.mid.gain.gain.cancelScheduledValues(now);
    _rLoop.low.gain.gain.setTargetAtTime(lowTarget * scale, now, fadeSec);
    _rLoop.mid.gain.gain.setTargetAtTime(midTarget * scale, now, fadeSec);
  }
  function _stopLoop(){
    if (!_rLoop) return;
    try {
      const now = ctx.currentTime;
      _rLoop.low.gain.gain.setTargetAtTime(0, now, 0.12);
      _rLoop.mid.gain.gain.setTargetAtTime(0, now, 0.12);
      const copy = _rLoop;
      setTimeout(()=>{
        try { copy.low.src.stop(); } catch(e){}
        try { copy.low.src.disconnect(); } catch(e){}
        try { copy.low.gain.disconnect(); } catch(e){}
        try { copy.mid.src.stop(); } catch(e){}
        try { copy.mid.src.disconnect(); } catch(e){}
        try { copy.mid.gain.disconnect(); } catch(e){}
        try { copy.mid.hp.disconnect(); } catch(e){}
      }, 360);
    } catch(e){}
    _rLoop = null;
    window[IP+'_rattle_state'].active = false;
    _clearTransientTimer();
  }
  // -------------------------
  // Transient scheduler & play
  // -------------------------
  function _scheduleNextTransient(){
    _clearTransientTimer();
    const interval = TRANSIENT_INTERVAL_MIN_MS + Math.random() * (TRANSIENT_INTERVAL_MAX_MS - TRANSIENT_INTERVAL_MIN_MS);
    _transTimer = setTimeout(()=> {
      _transTimer = null;
      _maybePlayTransient();
      if (_rLoop) _scheduleNextTransient();
    }, Math.round(interval));
  }
  function _clearTransientTimer(){ if (_transTimer) { clearTimeout(_transTimer); _transTimer = null; } }
  function _maybePlayTransient(){
    try {
      if (!_rLoop) return;
      const nowMs = Date.now();
      const key = 'transC';
      if (nowMs - (_transCooldown[key] || 0) < TRANSIENT_COOLDOWN_MS) return;
      _playTransient(key);
    } catch(e){ console.warn('[GE90 Rattle] maybePlayTransient error', e); }
  }
  function _playTransient(key){
    try {
      const buf = _buf[key];
      if (!buf) return;
      _transCooldown[key] = Date.now();
      const maxOffset = Math.max(0, Math.min(0.12, (buf.duration || 0) * 0.08));
      const startOffset = Math.random() * maxOffset;
      const gain = 0.16 + Math.random() * 0.26;
      const pitchJitter = 0.10 + Math.random()*0.06;
      _playOneShot(buf, gain, pitchJitter, startOffset, 1.24, 160);
    } catch(e){ console.warn('[GE90 Rattle] playTransient error', e); }
  }
  // -------------------------
  // Accel computation (hoisted)
  // -------------------------
  function _computeAccel(){
    const now = performance.now();
    const speedKts = _readGroundspeedKts();
    if (speedKts === null) return null;
    if (_lastSpeedKts === null) {
      _lastSpeedKts = speedKts;
      _lastSpeedTs = now;
      return { speedKts, accel: 0 };
    }
    const dt = Math.max(1, now - _lastSpeedTs) / 1000;
    const ds_kts = speedKts - _lastSpeedKts;
    const ds_mps = ds_kts * 0.514444;
    const accel = ds_mps / dt;
    _lastSpeedKts = speedKts;
    _lastSpeedTs = now;
    return { speedKts, accel };
  }
  // -------------------------
  // Main poll: uses groundContact if available, otherwise fallback
  // -------------------------
  function _startPoll(){
    if (_poll) return;
    _poll = setInterval(async ()=>{
      try {
        if (typeof window[IP+'_enabled'] !== 'undefined' && window[IP+'_enabled'] === false) return;
       // the viewType global is kept in sync by applyEffectiveMute every 250ms
            // Use it directly — getViewType is not accessible from this closure scope
            const view = window[IP+'_viewType'] || 'exterior';
        if (!/cockpit|jump/i.test(view) && view !== 'vc') { _stopLoop(); return; }
        // If geofs.aircraft.instance.groundContact exists, use it as authoritative gate
        const groundContactRaw = _readGroundContact(); // true/false/null
        let isGrounded = false;
        if (groundContactRaw === true || groundContactRaw === false) {
          isGrounded = groundContactRaw;
        } else {
          // fallback conservative test
          isGrounded = _isOnGroundFallback();
        }
        // If not grounded, stop immediately
        if (!isGrounded) {
          _stopLoop();
          return;
        }
        // If grounded, continue with spawn protection and intensity logic
        const now = performance.now();
        const sinceSpawn = (window[IP+'_spawnTime']) ? (now - window[IP+'_spawnTime']) : 999999;
        if (sinceSpawn < SPAWN_PROTECT_MS) return;
        const s = _computeAccel();
        if (!s) return;
        const speed = s.speedKts;
        const accel = s.accel;
        // Start/Update logic with gradual introduction
        if (speed >= SPEED_START_KTS && (accel >= ACCEL_START || speed >= SPEED_MIN_KTS)) {
          if (_accelStartTime === 0) _accelStartTime = now;
          if (now - _accelStartTime >= ACCEL_HOLD_MS) {
            const intensity = _computeIntensity(speed, accel);
            if (!_rLoop) {
              await _loadAll();
              _startLoop(intensity * 0.45 + 0.01);
            } else {
              _updateLoopIntensity(intensity, LOOP_FADE);
            }
            return;
          }
        } else {
          _accelStartTime = 0;
        }
        // Stop condition: require both speed and accel low to stop
        if (_rLoop && (speed <= SPEED_STOP_KTS && accel <= ACCEL_STOP)) {
          _stopLoop();
        }
      } catch(e){
        console.warn('[GE90 Rattle] poll error', e);
      }
    }, POLL_MS);
  }
  function _computeIntensity(speedKts, accel){
    const SPEED_FULL = 100;
    const speedBaseline = Math.max(0, Math.min(1, (speedKts - SPEED_START_KTS) / (SPEED_FULL - SPEED_START_KTS)));
    const accelIntensity = Math.max(0, Math.min(1, (accel - ACCEL_START) / (2.0 - ACCEL_START)));
    return Math.max(speedBaseline * 0.9, accelIntensity * 1.0);
  }
  // Start when audio context exists
  (function waitForReady(){
    if (typeof ctx !== 'undefined' && ctx && (_getOutNode())) {
      _startPoll();
    } else {
      setTimeout(waitForReady, 250);
    }
  })();
  // -------------------------
  // Test helpers
  // -------------------------
  window[NP+'_testRattleLoop'] = async function(intensity=1.0){
    await _loadAll();
    _startLoop(Math.max(0, Math.min(1, intensity)));
  };
  window[NP+'_stopRattleLoop'] = function(){ _stopLoop(); };
  window[NP+'_testRattleHit'] = async function(severity=1.0){
    await _loadAll();
    _playOneShot(_buf.low, 0.28 * severity, 0.02);
    setTimeout(()=>{ _playOneShot(_buf.midL, 0.44 * severity, 0.08, 0, 1.26); }, 20);
    setTimeout(()=>{ _playOneShot(_buf.transC, 0.40 * severity, 0.14, 0, 1.24, 160); }, 40);
  };
  window[NP+'_auditionRattleLow'] = async function(g=0.30){ await _loadAll(); _playOneShot(_buf.low, g, 0.01); };
  window[NP+'_auditionRattleMid'] = async function(g=0.48){ await _loadAll(); _playOneShot(_buf.midL, g, 0.10, 0, 1.26, 140); };
  window[NP+'_auditionRattleTransC'] = async function(g=0.44){ await _loadAll(); _playOneShot(_buf.transC, g, 0.14, 0, 1.24, 160); };
  // Cleanup
  window.addEventListener('beforeunload', ()=>{ try{ clearInterval(_poll); _clearTransientTimer(); }catch(e){} });
})();
// -------------------------------------------------------------
// TIRE NOISE (ground roll, speed-scaled volume, all exterior views)
// -------------------------------------------------------------
(function setupTireNoise(){
    // Tunables
    const POLL_MS           = 80;
    const SPEED_MIN_KTS     = 2;     // below this, silence
    const SPEED_MAX_KTS     = 100;   // at this speed, full volume
    const VOL_MAX           = 0.65;   // max gain at SPEED_MAX_KTS
    const SMOOTH_ALPHA      = 0.12;  // EMA smoothing on gain changes
    const XFADE_SEC         = 0.08;  // crossfade overlap between loop iterations
    const GROUND_DEBOUNCE   = 5;     // ticks of !groundContact before stopping (~400ms)
    // Audible in all views EXCEPT interior/cockpit views
    const EXCLUDED_VIEWS = ['cockpit', 'jumpseat', 'wing2'];
    let tireBuf     = null;
    let tireGain    = null;
    let smoothedGain = 0;
    let isPlaying   = false;
    // -----------------------------------------------------------
    // Seamless loop buffer — crossfades tail into head
    // -----------------------------------------------------------
    function createSeamlessTireBuf(srcBuf, fadeSec = 0.03){
        const sr = srcBuf.sampleRate;
        const fadeSamples = Math.floor(fadeSec * sr);
        const length = srcBuf.length;
        if (fadeSamples * 2 >= length) return srcBuf;
        const out = ctx.createBuffer(srcBuf.numberOfChannels, length, sr);
        for (let ch = 0; ch < srcBuf.numberOfChannels; ch++){
            const inData  = srcBuf.getChannelData(ch);
            const outData = out.getChannelData(ch);
            for (let i = 0; i < length; i++) outData[i] = inData[i];
            for (let i = 0; i < fadeSamples; i++){
                const fadeOut = 1 - (i / fadeSamples);
                const fadeIn  = i / fadeSamples;
                const tailIdx = length - fadeSamples + i;
                outData[tailIdx] = inData[tailIdx] * fadeOut + inData[i] * fadeIn;
            }
        }
        return out;
    }
    // -----------------------------------------------------------
    // Load + process buffer once
    // -----------------------------------------------------------
    async function loadTireBuf(){
        if (tireBuf) return tireBuf;
        try {
            const ab  = await fetch(B.assets.tire, { mode: 'cors' }).then(r => r.arrayBuffer());
            const raw = await ctx.decodeAudioData(ab);
            tireBuf   = createSeamlessTireBuf(raw, 0.03);
        } catch(e){
            console.warn('[GE90 Tire] load failed', e);
        }
        return tireBuf;
    }
    // -----------------------------------------------------------
    // Crossfade looping — overlapping one-shots, no loop seam click
    // -----------------------------------------------------------
    function scheduleTireLoop(){
        if (!tireBuf || !tireGain) return;
        const bufDur = tireBuf.duration;
        function scheduleNext(playAt){
            if (!isPlaying || !tireGain) return;
            const src   = ctx.createBufferSource();
            src.buffer  = tireBuf;
            src._GE90_TAG = true;
            const xGain = ctx.createGain();
            xGain._GE90_TAG = true;
            xGain.gain.setValueAtTime(0, playAt);
            xGain.gain.linearRampToValueAtTime(1, playAt + XFADE_SEC);
            xGain.gain.setValueAtTime(1, playAt + bufDur - XFADE_SEC);
            xGain.gain.linearRampToValueAtTime(0, playAt + bufDur);
            src.connect(xGain);
            xGain.connect(tireGain);
            src.start(playAt);
            const nextStartAt  = playAt + bufDur - XFADE_SEC;
            const timeUntilNext = (nextStartAt - ctx.currentTime) * 1000;
            setTimeout(() => scheduleNext(nextStartAt), Math.max(0, timeUntilNext - 50));
        }
        scheduleNext(ctx.currentTime);
    }
    // -----------------------------------------------------------
    // Start / stop
    // -----------------------------------------------------------
    function startTire(){
        if (isPlaying || !tireBuf) return;
        tireGain = ctx.createGain();
        tireGain._GE90_TAG = true;
        tireGain.gain.setValueAtTime(0, ctx.currentTime);
        tireGain.connect(master);
        window[IP+'_tireGain'] = tireGain;
        isPlaying = true;
        scheduleTireLoop();
    }
    function stopTire(){
        if (!isPlaying) return;
        isPlaying    = false;
        smoothedGain = 0;
        try {
            tireGain.gain.cancelAndHoldAtTime(ctx.currentTime);
            tireGain.gain.setTargetAtTime(0, ctx.currentTime, 0.2);
            setTimeout(() => {
                try { tireGain.disconnect(); } catch(e){}
                tireGain = null;
                window[IP+'_tireGain'] = null;
            }, 600);
        } catch(e){}
    }
    // -----------------------------------------------------------
    // Main poll
    // -----------------------------------------------------------
    setInterval(async () => {
        try {
            const effectiveMuted = window[IP+'_persistentlyMuted'] ||
                                   window[IP+'_userMuted'] ||
                                   window[IP+'_paused'];
            const camView  = window.geofs?.camera?.currentView || '';
            const viewType = (typeof getViewType === 'function')
                ? getViewType()
                : (window[IP+'_viewType'] || camView || 'exterior');
            const inst       = window.geofs?.aircraft?.instance;
            const isGrounded = !!(inst?.groundContact);
            // Initialize debounce counter
            if (!window[IP+'_tireGroundTicks']) window[IP+'_tireGroundTicks'] = 0;
            // Stop immediately on mute or excluded view
            if (effectiveMuted || EXCLUDED_VIEWS.includes(viewType)) {
                window[IP+'_tireGroundTicks'] = 0;
                if (isPlaying) stopTire();
                return;
            }
            // Debounce ground contact loss to swallow terrain glitch flickers
            if (!isGrounded) {
                window[IP+'_tireGroundTicks']++;
                if (window[IP+'_tireGroundTicks'] >= GROUND_DEBOUNCE) {
                    // Sustained airborne — genuine liftoff
                    if (isPlaying) stopTire();
                }
                // Still within debounce window — keep playing
                return;
            }
            // Confirmed grounded — reset debounce counter
            window[IP+'_tireGroundTicks'] = 0;
            // Read ground speed (m/s → knots)
            const speedMps = inst?.groundSpeed ?? inst?.velocityScalar ?? 0;
            const speedKts = speedMps * 1.94384;
            // Compute target gain
            const t = Math.max(0, Math.min(1,
                (speedKts - SPEED_MIN_KTS) / (SPEED_MAX_KTS - SPEED_MIN_KTS)
            ));
            const targetGain = t * VOL_MAX;
            // Don't start until there's meaningful speed
            if (targetGain < 0.001 && !isPlaying) return;
            // Load buffer if not yet ready
            if (!tireBuf) { await loadTireBuf(); return; }
            // Start playback if not already running
            if (!isPlaying) startTire();
            // Smooth gain update — cancel previous ramp first to avoid clicks
            smoothedGain += (targetGain - smoothedGain) * SMOOTH_ALPHA;
            if (tireGain) {
                tireGain.gain.cancelAndHoldAtTime(ctx.currentTime);
                tireGain.gain.setTargetAtTime(smoothedGain, ctx.currentTime, 0.15);
            }
        } catch(e){}
    }, POLL_MS);
})();
        // -------------------------
        // Dual + exterior ambience loader
        // -------------------------
        (async function setupAmbiences(){
          try {
            if (!window[IP+'_ambienceCockpitA'] ||
                !window[IP+'_ambienceCockpitB'] ||
                !window[IP+'_ambienceWing2'] ||
                !window[IP+'_ambienceWing2Air'] ||
                !window[IP+'_ambienceWing2AirB'] ||
                !window[IP+'_ambienceExterior']) {
              const createSeamlessBuffer = (srcBuf, fadeSec = 1.2) => {
    const sr = srcBuf.sampleRate;
    const length = srcBuf.length;
    // Adaptive cap: never let the crossfade eat more than ~25% of the loop,
    // so shorter ambience clips still get the biggest smooth fade they can
    // support instead of silently falling back to a hard, un-crossfaded loop.
    const maxFadeSec = (length / sr) * 0.25;
    const effectiveFadeSec = Math.min(fadeSec, maxFadeSec);
    const fadeSamples = Math.floor(effectiveFadeSec * sr);
    if (fadeSamples < 8) return srcBuf;
    const out = ctx.createBuffer(srcBuf.numberOfChannels, length, sr);
    for (let ch = 0; ch < srcBuf.numberOfChannels; ch++) {
        const inData = srcBuf.getChannelData(ch);
        const outData = out.getChannelData(ch);
        for (let i = 0; i < length; i++) outData[i] = inData[i];
        for (let i = 0; i < fadeSamples; i++) {
            // Equal-power crossfade (not linear): linear fades dip in perceived
            // loudness mid-transition on broadband ambience/noise material.
            const x = i / fadeSamples;
            const fadeOut = Math.cos(x * Math.PI / 2);
            const fadeIn  = Math.sin(x * Math.PI / 2);
            const tailIdx = length - fadeSamples + i;
            outData[tailIdx] = inData[tailIdx] * fadeOut + inData[i] * fadeIn;
        }
    }
    return out;
};
const loadAmbience = async (url, options = {}) => {
    const { offset = 0, jitter = false, lpfFreq = null } = options;
    const rawBuf = await fetch(url, { mode: 'cors' })
      .then(r => r.arrayBuffer())
      .then(ab => ctx.decodeAudioData(ab));
    const buf = createSeamlessBuffer(rawBuf, 1.2); // ~1.2s equal-power crossfade at loop seam (was 40ms linear)
                const ambGain = ctx.createGain();
                ambGain.gain.setValueAtTime(0, ctx.currentTime);
                ambGain._GE90_TAG = true;
                let nodeChainHead = ambGain;
                let lpf = null;
                if (lpfFreq) {
                  lpf = ctx.createBiquadFilter();
                  lpf.type = "lowpass";
                  lpf.frequency.setValueAtTime(lpfFreq, ctx.currentTime);
                  lpf.Q.setValueAtTime(0.7, ctx.currentTime);
                  lpf._GE90_TAG = true;
                  ambGain.connect(lpf);
                  nodeChainHead = lpf;
                }
                nodeChainHead.connect(master);
                const ambSrc = ctx.createBufferSource();
                ambSrc.buffer = buf;
                ambSrc.loop = true;
                ambSrc._GE90_TAG = true;
                ambSrc.connect(ambGain);
                ambSrc.start(ctx.currentTime + offset);
                let jitterTimer = null;
                if (jitter) {
                  jitterTimer = setInterval(()=>{
                    try {
                      const base = 1.0;
                      const delta = (Math.random() * 0.006) - 0.003;
                      ambSrc.playbackRate.setValueAtTime(base + delta, ctx.currentTime);
                    } catch(e){}
                  }, 8000 + Math.random()*4000);
                }
                return { ambSrc, ambGain, jitterTimer, lpf };
              };
              window[IP+'_ambienceCockpitA'] = await loadAmbience(B.assets.ambCockpit, {
                offset: 0.0,
                jitter: true
              });
              window[IP+'_ambienceCockpitB'] = await loadAmbience(B.assets.ambCockpit, {
                offset: 1.5 + Math.random()*2.5,
                jitter: true
              });
              window[IP+'_ambienceWing2'] = await loadAmbience(B.assets.ambWing2, {
                offset: 0.0,
                jitter: true
              });
window[IP+'_ambienceWing2Air'] = await loadAmbience(B.assets.ambWing2Air, {
                offset: 0.0,
                jitter: true
              });
              window[IP+'_ambienceWing2AirB'] = await loadAmbience(B.assets.ambWing2Air, {
                offset: 1.5 + Math.random()*2.5,
                jitter: true
              });
              window[IP+'_ambienceExterior'] = await loadAmbience(B.assets.ambExterior, {
                offset: 0.0,
                jitter: true,
                lpfFreq: B.mix.exteriorLpfFreq
              });
            }
          } catch(e){
            console.warn("[GE90] ambience load failed", e);
          }
        })();
// -------------------------------------------------------------
// TAKEOFF CONFIG WARNING (max thrust + no flaps out, on ground) — looping alarm
// -------------------------------------------------------------
(function setupConfigWarningAlarm(){
    const ctx = window[IP+'_audioCtx'];
    const master = window[IP+'_master'];
    if (!ctx || !master) return;
    const THROTTLE_MAX_THRESHOLD = 0.97; // treat this as "max/takeoff thrust applied"
    const FLAPS_OUT_THRESHOLD    = 0.01; // anything above this counts as flaps out
    const POLL_MS   = 150;
    const FADE_IN   = 0.15;
    const FADE_OUT  = 0.25;
    let alarmBuf     = null;
    let alarmLoading = false;
    let alarmSrc     = null;
    let alarmGain    = null;
    let alarmActive  = false;
    async function loadAlarmBuffer(){
        if (alarmBuf || alarmLoading) return;
        alarmLoading = true;
        try {
            const r = await fetch(B.assets.configAlarm, { mode: "cors" });
            if (!r.ok) throw new Error(`HTTP ${r.status}`);
            const ab = await r.arrayBuffer();
            const decoded = await ctx.decodeAudioData(ab);
            alarmBuf = window._GE_seamlessLoopBuffer(ctx, decoded, 0.05);
        } catch(e){
            console.warn("[GE90 ConfigWarning] buffer load failed", e);
        } finally {
            alarmLoading = false;
        }
    }
    loadAlarmBuffer();
    function startAlarm(){
        if (alarmActive || !alarmBuf) return;
        const src = ctx.createBufferSource();
        src.buffer = alarmBuf;
        src.loop = true;
        src._GE90_TAG = true;
        const g = ctx.createGain();
        g.gain.setValueAtTime(0, ctx.currentTime);
        g._GE90_TAG = true;
        src.connect(g);
        g.connect(master);
        src.start(0);
        g.gain.linearRampToValueAtTime(1.0, ctx.currentTime + FADE_IN);
        alarmSrc  = src;
        alarmGain = g;
        alarmActive = true;
    }
    function stopAlarm(){
        if (!alarmActive) return;
        try {
            const src = alarmSrc;
            const g   = alarmGain;
            alarmActive = false;
            if (g) {
                g.gain.cancelScheduledValues(ctx.currentTime);
                g.gain.setValueAtTime(g.gain.value, ctx.currentTime);
                g.gain.linearRampToValueAtTime(0, ctx.currentTime + FADE_OUT);
            }
            setTimeout(() => {
                try { if (src) { src.stop(0); src.disconnect(); } } catch(e){}
                try { if (g) g.disconnect(); } catch(e){}
            }, (FADE_OUT * 1000) + 60);
            alarmSrc  = null;
            alarmGain = null;
        } catch(e){
            console.warn("[GE90 ConfigWarning] stopAlarm failed", e);
        }
    }
    setInterval(async () => {
        try {
            if (!window[IP+'_active']) { stopAlarm(); return; }
            const effectiveMuted =
                !!window[IP+'_userMuted'] ||
                !!window[IP+'_siteMuted'] ||
                !!window[IP+'_paused'] ||
                !!window[IP+'_persistentlyMuted'];
            const view = (typeof getViewType === "function")
                ? getViewType()
                : (window[IP+'_viewType'] || "exterior");
            const inst = window.geofs?.aircraft?.instance;
            const grounded = !!(inst?.groundContact);
            const throttle = window.geofs?.animation?.values?.throttle;
            const flapsPosition = window.geofs?.animation?.values?.flapsPosition;
            const maxThrust = (typeof throttle === "number") && throttle >= THROTTLE_MAX_THRESHOLD;
            const noFlapsOut = (typeof flapsPosition === "number") && flapsPosition <= FLAPS_OUT_THRESHOLD;
            const shouldPlay = grounded && maxThrust && noFlapsOut && view === "cockpit" && !effectiveMuted;
            if (shouldPlay) {
                if (!alarmBuf) { await loadAlarmBuffer(); return; }
                startAlarm();
            } else {
                stopAlarm();
            }
        } catch(e){
            console.warn("[GE90 ConfigWarning] poll error", e);
        }
    }, POLL_MS);
})();
// -------------------------------------------------------------
// RAIN SOUND SYSTEM (cockpit-only, fixed volume, GE90 interior chain)
// -------------------------------------------------------------
(function setupRainSound(){
    const ctx = window[IP+'_audioCtx'];
    const finalMaster = window[IP+'_finalMaster'] || window[IP+'_master'];
    if (!ctx || !finalMaster) return;
    let rainBuf = null;
    let rainSrc = null;
    let rainGain = null;
    // -----------------------------
    // Load rain buffer
    // -----------------------------
    async function loadRainBuffer(){
        if (rainBuf) return rainBuf;
        try {
            const r = await fetch(B.assets.rain, { mode: "cors" });
            const ab = await r.arrayBuffer();
            rainBuf = await ctx.decodeAudioData(ab);
        } catch(e){
            console.warn("[GE90 Rain] load failed", e);
        }
        return rainBuf;
    }
    // -----------------------------
    // Start rain loop
    // -----------------------------
    function startRain(){
        if (rainSrc || !rainBuf) return;
        rainSrc = ctx.createBufferSource();
        rainSrc.buffer = window._GE_seamlessLoopBuffer(ctx, rainBuf, 1.0);
        rainSrc.loop = true;
        rainSrc._GE90_TAG = true;
        rainGain = ctx.createGain();
        rainGain.gain.value = 0;
        rainGain._GE90_TAG = true;
        rainSrc.connect(rainGain);
        rainGain.connect(finalMaster);
        // register as interior node if array exists
        if (Array.isArray(window[IP+'_interior_nodes'])) {
            window[IP+'_interior_nodes'].push(rainGain);
        }
        rainSrc.start(0);
    }
    // -----------------------------
    // Stop rain loop
    // -----------------------------
    function stopRain(){
        if (!rainSrc) return;
        try {
            rainGain.gain.setTargetAtTime(0, ctx.currentTime, 0.25);
            setTimeout(() => {
                try { rainSrc.stop(); } catch(e){}
                rainSrc = null;
                rainGain = null;
            }, 400);
        } catch(e){}
    }
    // -----------------------------
    // Rain ON/OFF detector
    // -----------------------------
    function isRaining(){
        try {
            const p = geofs?.fx?.precipitation;
            if (!p) return false;
            // GeoFS 4.0: `visible` stays true regardless of whether it's actually raining
            // (it just marks the precipitation system as loaded), but `type` only exists on
            // this object while precipitation is actively happening, and reads "rain"
            // specifically for rain (vs. snow) -- confirmed via live inspection.
            return p.type === "rain";
        } catch(e){
            return false;
        }
    }
    // -----------------------------
    // Cockpit view detector
    // -----------------------------
    function isCockpitView(){
        try {
            const v = String(geofs.camera.currentView || "").toLowerCase();
            return (
                v.includes("cockpit") ||
                v.includes("pilot") ||
                v.includes("interior")
            );
        } catch(e){
            return false;
        }
    }
    // -----------------------------
    // Main controller
    // -----------------------------
    setInterval(async () => {
        try {
            const cockpitView = isCockpitView();
            const muted =
                window[IP+'_persistentlyMuted'] ||
                window[IP+'_userMuted'] ||
                window[IP+'_paused'] ||
                window[IP+'_muted'] ||
                window[IP+'_siteMuted'];
            if (!rainBuf) await loadRainBuffer();
            const raining = isRaining();
            if (cockpitView && !muted && raining) {
                startRain();
                if (rainGain) {
                    const targetGain = 0.35; // adjust to taste
                    rainGain.gain.setTargetAtTime(targetGain, ctx.currentTime, 0.20);
                }
            } else {
                stopRain();
            }
        } catch(e){
            console.warn("[GE90 Rain] controller error", e);
        }
    }, 200);
})();
        // -------------------------
        // Site mute detection
        // -------------------------
        function detectSiteMuted(){
          try {
            if (window.Howler && typeof window.Howler._muted !== 'undefined')
              return !!window.Howler._muted;
            const medias = Array.from(document.querySelectorAll('audio,video'));
            const userMutedMedias = medias.filter(m => !m._GE90_FORCED_MUTE_BY_PACK && !m._GE_safetyViewGated);
            if (!userMutedMedias.length) return false;
            return userMutedMedias.every(m => m.muted || m.volume === 0);
          } catch(e){
            return false;
          }
        }
// -----------------------------
// Engine on/off toggle sound (state-based, cockpit only)
// -----------------------------
(function(){
  const POLL_MS = 120;
  const DEBOUNCE_MS = 500;
  const SPAWN_PROTECT_MS = window.SPAWN_PROTECTION_MS || 5000;
  let _et_buf = null;
  let _et_poll = null;
  let _et_lastState = null;
  let _et_lastToggle = 0;
  async function _et_load(){
    if (_et_buf) return _et_buf;
    try {
      const r = await fetch(B.assets.engineToggle, { mode: 'cors' });
      const ab = await r.arrayBuffer();
      _et_buf = await ctx.decodeAudioData(ab);
    } catch (e) {
      console.warn("[GE90 EngineToggle] load failed", e);
    }
    return _et_buf;
  }
  function _et_playOnce(gain = 0.6){
    try {
      if (!ctx || !_et_buf) return;
      // Only play in cockpit / jump seat
      const view = (typeof getViewType === 'function') ? getViewType() : (window[IP+'_viewType'] || 'exterior');
      if (view !== 'cockpit' && !/jump/i.test(view) && view !== 'vc') return;
      // Create a clean one-shot that avoids jitter and heavy EQ
      const src = ctx.createBufferSource();
      src.buffer = _et_buf;
      src.loop = false;
      src.playbackRate.setValueAtTime(1.0, ctx.currentTime);
      src._GE90_TAG = true;
      const g = ctx.createGain();
      g.gain.setValueAtTime(gain, ctx.currentTime);
      g._GE90_TAG = true;
      src.connect(g);
      // Prefer connecting to finalMaster (post-processing) so the sound is not altered by interior EQ chain
      // finalMaster exists in your script as window[IP+'_finalMaster']; fallback to master
      const outNode = window[IP+'_finalMaster'] || master || ctx.destination;
      g.connect(outNode);
      src.start(ctx.currentTime);
      // Cleanup after playback
      const cleanupMs = Math.round((_et_buf.duration || 2) * 1000 + 200);
      setTimeout(()=>{
        try { src.stop(); } catch(e){}
        try { src.disconnect(); } catch(e){}
        try { g.disconnect(); } catch(e){}
      }, cleanupMs);
    } catch (e) {
      console.warn("[GE90 EngineToggle] play error", e);
    }
  }
  // Robust read of engine boolean state (tries multiple properties)
  function _readEngineState(){
    try {
      const inst = window.geofs?.aircraft?.instance;
      if (!inst) return null;
      const engine = inst.engine || inst.engines?.[0] || null;
      if (!engine) return null;
      // Candidate boolean/numeric properties
      const candidates = [
        engine.on,
        engine.running,
        engine.isRunning,
        engine.master,
        engine.starter,
        engine.starterOn,
        engine.runningState,
        engine.state
      ];
      for (const c of candidates) {
        if (typeof c === 'boolean') return c;
        if (typeof c === 'number') return c > 0;
        if (typeof c === 'string' && (c === 'on' || c === 'running' || c === '1')) return true;
      }
      // Fallback: if rpm exists, treat rpm > small threshold as running
      if (typeof engine.rpm === 'number' && !isNaN(engine.rpm)) {
        return engine.rpm > 50;
      }
      return null;
    } catch(e){
      return null;
    }
  }
  function _et_start(){
    if (_et_poll) return;
    _et_poll = setInterval(async ()=>{
      try {
        const now = performance.now();
        const timeSinceSpawn = (window[IP+'_spawnTime']) ? (now - window[IP+'_spawnTime']) : 0;
        const spawnProtected = timeSinceSpawn < SPAWN_PROTECT_MS;
        const state = _readEngineState();
        if (state === null) return;
        // initialize
        if (_et_lastState === null) {
          _et_lastState = state;
          return;
        }
        // Detect change (toggle)
        if (state !== _et_lastState && (now - _et_lastToggle) > DEBOUNCE_MS && !spawnProtected) {
          _et_lastToggle = now;
          await _et_load();
          // Play only once per toggle
          _et_playOnce(0.4);
        }
        _et_lastState = state;
      } catch(e){
        console.warn("[GE90 EngineToggle] poll error", e);
      }
    }, POLL_MS);
  }
  // Wait for audio context and start
  (function waitForReady(){
    if (typeof ctx !== 'undefined' && ctx && (window[IP+'_finalMaster'] || master)) {
      _et_start();
    } else {
      setTimeout(waitForReady, 250);
    }
  })();
  // Test helper
  window[NP+'_testEngineToggleSound'] = async function(){
    await _et_load();
    _et_playOnce(0.6);
  };
})();
        // -------------------------------------------------------------
        // FLAP SOUNDS - Threshold-based with continuous hum during movement
        // -------------------------------------------------------------
        (function setupFlapSounds(){
          const ctx = window[IP+'_audioCtx'];
          const master = window[IP+'_master'];
          if (!ctx || !master) return;
          let lastFlap = null;
          let lastMovementTime = 0; // wall-clock time (ms) of the last detected flap movement
          let clickBuf = null;
          let humBuf = null;
          let humSrc = null;
          let humGain = null;
          let clickDebounceTime = 0; // Add click debounce
          // One flap key press = one click, debounced: a change only counts as
          // new if flapsTarget had been idle for this long first (a wobble right
          // after just extends the wait). Tune up if clicks double, down if
          // quick taps get swallowed.
          const FLAP_CLICK_HOLDOFF_MS = 300;
          const FLAP_TARGET_EPS = 0.0005;        // flapsTarget wiggle smaller than this isn't a command
          const FLAP_CLICK_STEAL_FADE = 0.008;   // s — quick fade when a new click replaces one still sounding
          let clickVoice = null; // the single click currently sounding: a retrigger replaces it, never layers on it
          async function loadBuffers(){
            try {
              if (!clickBuf) {
                try {
                  const r = await fetch(B.assets.flapClick, {mode:"cors"});
                  if (!r.ok) throw new Error(`HTTP ${r.status}`);
                  const ab = await r.arrayBuffer();
                  clickBuf = await ctx.decodeAudioData(ab);
                } catch(e) {
                  console.warn("[GE90 Flap] Click buffer failed, creating fallback:", e);
                  // Create fallback click buffer
                  clickBuf = createClickFallback();
                }
              }
              if (!humBuf) {
                try {
                  const r = await fetch(B.assets.flapHum, {mode:"cors"});
                  if (!r.ok) throw new Error(`HTTP ${r.status}`);
                  const ab = await r.arrayBuffer();
                  humBuf = await ctx.decodeAudioData(ab);
                } catch(e) {
                  console.warn("[GE90 Flap] Hum buffer failed, creating fallback:", e);
                  // Create fallback hum buffer
                  humBuf = createHumFallback();
                }
              }
              await loadMotorBuffers();
            } catch(e){
              console.warn("[GE90 Flap] Buffer load failed:", e);
            }
          }
          function createClickFallback(){
            const duration = 0.1; // 100ms click
            const sampleRate = ctx.sampleRate;
            const frames = duration * sampleRate;
            const buffer = ctx.createBuffer(1, frames, sampleRate);
            const data = buffer.getChannelData(0);
            // Create a sharp click sound
            for (let i = 0; i < frames; i++) {
              if (i < 100) {
                data[i] = (Math.random() - 0.5) * 0.5;
              } else {
                data[i] = 0;
              }
            }
            return buffer;
          }
          function createHumFallback(){
            const duration = 2.0; // 2 second loop
            const sampleRate = ctx.sampleRate;
            const frames = duration * sampleRate;
            const buffer = ctx.createBuffer(1, frames, sampleRate);
            const data = buffer.getChannelData(0);
            // Create a low-frequency hum
            for (let i = 0; i < frames; i++) {
              const t = i / sampleRate;
              // Mix of low frequencies for motor hum
              data[i] = (Math.sin(2 * Math.PI * 60 * t) * 0.1 +    // 60Hz
                        Math.sin(2 * Math.PI * 120 * t) * 0.05 +   // 120Hz
                        Math.sin(2 * Math.PI * 180 * t) * 0.03 +   // 180Hz
                        (Math.random() - 0.5) * 0.02) * 0.3;      // Noise
            }
            return buffer;
          }
          function playClick(){
            try {
              if (!clickBuf) return;
              // A suspended context (sim paused, or this pack's aircraft isn't
              // the active one) doesn't play anything — it just queues start(0)
              // and then fires every queued click at once on resume. Skip.
              if (ctx.state !== 'running') return;
              const t = ctx.currentTime;
              // FMOD-style "max instances = 1, steal oldest": if a click is
              // still sounding, fade it out in a few ms and replace it, so two
              // triggers can never sit on top of each other as an echo.
              if (clickVoice) {
                fadeOutVoice(clickVoice, t, FLAP_CLICK_STEAL_FADE);
                clickVoice = null;
              }
              const src = ctx.createBufferSource();
              src.buffer = clickBuf;
              src._GE90_TAG = true;
              const g = ctx.createGain();
              g.gain.value = 1.35; // Increased from 1.0 (flap click louder)
              g._GE90_TAG = true;
              src.connect(g);
              g.connect(master);
              const voice = { src, gain: g };
              src.onended = () => { if (clickVoice === voice) clickVoice = null; };
              clickVoice = voice;
              src.start(t);
            } catch(e){
              console.warn("[GE90 Flap] Click failed", e);
            }
          }
          // THREE-STAGE FLAP MOTOR
          // A bank with B.assets.flapMotor gets start->loop->end (crossfaded)
          // instead of one abrupt loop; end clip plays out fully. Banks
          // without it keep the plain hum below, unchanged.
          let motorBufs   = null;      // { start, loop, end }
          let motorState  = 'idle';    // idle | starting | looping | ending
          let motorVoice_ = null;      // { src, gain } currently sounding
          let motorTimer  = null;      // scheduled handoff
          let motorVol    = 0.6;
          let motorFailed = false;
          const MOTOR_XFADE = (B.assets.flapMotor && B.assets.flapMotor.crossfade) || 0.12;
          function motorEnabled(){ return !!(B.assets.flapMotor && motorBufs); }
          async function loadMotorBuffers(){
            const cfg = B.assets.flapMotor;
            if (!cfg || motorBufs || motorFailed) return;
            try {
              const [s, l, e] = await Promise.all(['start','loop','end'].map(async k => {
                const r = await fetch(cfg[k], { mode: 'cors' });
                if (!r.ok) throw new Error(k + ' HTTP ' + r.status);
                return ctx.decodeAudioData(await r.arrayBuffer());
              }));
              motorBufs = {
                start: s,
                // Same tail-into-head crossfade the engine beds use, so the
                // middle leg can hold indefinitely without a seam tick.
                loop:  window._GE_seamlessLoopBuffer(ctx, l, 0.2),
                end:   e
              };
            } catch(err){
              // Fall back to the single-loop hum, which is already loaded.
              motorFailed = true;
              console.warn('[GEFMOD Flap] motor clips failed, using single-loop hum', err);
            }
          }
          function newMotorVoice(buf, loop){
            const src = ctx.createBufferSource();
            src.buffer = buf;
            src.loop = !!loop;
            src._GE90_TAG = true;
            const g = ctx.createGain();
            g.gain.value = 0;
            g._GE90_TAG = true;
            src.connect(g);
            g.connect(master);
            return { src, gain: g };
          }
          function fadeOutVoice(v, at, dur){
            if (!v) return;
            try {
              v.gain.gain.cancelScheduledValues(at);
              v.gain.gain.setValueAtTime(v.gain.gain.value, at);
              v.gain.gain.linearRampToValueAtTime(0, at + dur);
              v.src.stop(at + dur + 0.02);
            } catch(e){}
          }
          function clearMotorTimer(){
            if (motorTimer) { clearTimeout(motorTimer); motorTimer = null; }
          }
          function motorBeginLoop(at, prev){
            const v = newMotorVoice(motorBufs.loop, true);
            v.gain.gain.setValueAtTime(0, at);
            v.gain.gain.linearRampToValueAtTime(motorVol, at + MOTOR_XFADE);
            v.src.start(at);
            fadeOutVoice(prev, at, MOTOR_XFADE);
            motorVoice_ = v;
            humSrc = v.src;          // keeps the existing poll guards working
            motorState = 'looping';
          }
          function motorStart(volume){
            motorVol = volume;
            if (motorState === 'starting' || motorState === 'looping') {
              // Already running — just retarget the level (view change).
              if (motorVoice_) motorVoice_.gain.gain.setTargetAtTime(motorVol, ctx.currentTime, 0.08);
              return;
            }
            if (motorState === 'ending') {
              // Movement resumed mid-tail: slide straight back into the loop
              // rather than replaying the start clip.
              clearMotorTimer();
              motorBeginLoop(ctx.currentTime, motorVoice_);
              return;
            }
            const t0 = ctx.currentTime;
            const v = newMotorVoice(motorBufs.start, false);
            v.gain.gain.setValueAtTime(0, t0);
            v.gain.gain.linearRampToValueAtTime(motorVol, t0 + 0.04);
            v.src.start(t0);
            motorVoice_ = v;
            humSrc = v.src;
            motorState = 'starting';
            // Hand off just before the start clip runs out so the two
            // overlap by one crossfade instead of butting together.
            const handoff = Math.max(0.05, motorBufs.start.duration - MOTOR_XFADE);
            clearMotorTimer();
            motorTimer = setTimeout(() => {
              motorTimer = null;
              if (motorState !== 'starting') return;
              motorBeginLoop(ctx.currentTime, motorVoice_);
            }, handoff * 1000);
          }
          function motorStop(){
            if (motorState === 'idle' || motorState === 'ending') return;
            clearMotorTimer();
            const at = ctx.currentTime;
            const prev = motorVoice_;
            const v = newMotorVoice(motorBufs.end, false);
            v.gain.gain.setValueAtTime(0, at);
            v.gain.gain.linearRampToValueAtTime(motorVol, at + MOTOR_XFADE);
            v.src.start(at);
            fadeOutVoice(prev, at, MOTOR_XFADE);
            motorVoice_ = v;
            motorState = 'ending';
            // Released here, not when the tail finishes, so movement that
            // resumes during the end clip re-enters through motorStart().
            humSrc = null;
            motorTimer = setTimeout(() => {
              motorTimer = null;
              if (motorState !== 'ending') return;
              try { v.src.stop(); } catch(e){}
              if (motorVoice_ === v) motorVoice_ = null;
              motorState = 'idle';
            }, (motorBufs.end.duration + 0.1) * 1000);
          }
          function startHum(volume = 0.3){
            if (motorEnabled()) return motorStart(volume);
            try {
              if (!humBuf) return;
              // Stop existing hum if running
              stopHum();
              humSrc = ctx.createBufferSource();
              humSrc.buffer = window._GE_seamlessLoopBuffer(ctx, humBuf, 0.2);
              humSrc.loop = true; // Continuous loop during movement
              humSrc._GE90_TAG = true;
              humGain = ctx.createGain();
              humGain.gain.value = 0;
              humGain._GE90_TAG = true;
              humSrc.connect(humGain);
              humGain.connect(master);
              humSrc.start(0);
              // Fade in with specified volume
              const now = ctx.currentTime;
              humGain.gain.setTargetAtTime(volume, now, 0.1);
            } catch(e){
              console.warn("[GE90 Flap] Hum start failed", e);
            }
          }
          function stopHum(){
            if (motorEnabled()) return motorStop();
            if (humSrc) {
              try {
                const now = ctx.currentTime;
                if (humGain) {
                  humGain.gain.setTargetAtTime(0, now, 0.2); // Fade out
                }
                setTimeout(() => {
                  try {
                    if (humSrc) {
                      humSrc.stop();
                      humSrc = null;
                    }
                  } catch(e) {}
                }, 300);
              } catch(e) {}
            }
          }
          function getViewType(){
  try {
    const cam = window.geofs?.camera;
    if (!cam) return "exterior";
    let s = cam.currentModeName || cam.currentView || cam.currentDefinition?.name || cam.mode || "";
    s = String(s).toLowerCase();
    if (!s) return "exterior";
    // Was reading B.views2, a duplicate view-keyword table left unpopulated
    // (hasWingEngine: false, empty side/engine arrays) for almost every
    // pack, so this module could essentially never detect a wingEngine
    // view and the flap hum stayed silent there. B.views is the same
    // table the main engine-mix view detection already uses and is
    // correctly filled in per pack, so flap hum now agrees with it.
    const V = B.views;
    if (V.cockpit.some(k => s.includes(k))) return "cockpit";
    if (V.hasWingEngine &&
        V.side.some(k => s.includes(k)) &&
        V.engine.some(k => s.includes(k))) return "wingEngine";
    if (V.wing2.some(k => s.includes(k))) return "wing2";
    return "exterior";
  } catch(e){
    return "exterior";
  }
}
          // Listen for flap COMMAND events using flapsTarget monitoring
          function setupFlapCommandListener(){
            let lastFlapsTarget = null;
            let lastTargetChangeTime = 0; // wall-clock ms of the most recent flapsTarget change
            // Monitor flapsTarget value changes
            setInterval(() => {
              try {
                const currentFlapsTarget = geofs?.animation?.values?.flapsTarget;
                if (currentFlapsTarget === undefined || currentFlapsTarget === null) {
                  return; // No value yet
                }
                // Initialize on first run
                if (lastFlapsTarget === null) {
                  lastFlapsTarget = currentFlapsTarget;
                  return;
                }
                // Ignore float wiggle — only a real move of the target is a command
                if (Math.abs(currentFlapsTarget - lastFlapsTarget) < FLAP_TARGET_EPS) {
                  return;
                }
                // A real command arrives after the target has been sitting
                // still. Further changes hot on its heels (a ramp, a second
                // update of the same press) are the same command, not a new
                // one — they only push the "still" clock forward.
                const now = performance.now();
                const wasSettled = (now - lastTargetChangeTime) >= FLAP_CLICK_HOLDOFF_MS;
                lastTargetChangeTime = now;
                lastFlapsTarget = currentFlapsTarget;
                if (wasSettled) handleFlapCommand();
              } catch(e) {
              }
            }, 50); // Check frequently for responsive detection
          }
          async function handleFlapCommand(){
            // Only the pack that owns the current aircraft clicks; the others
            // keep their listeners running but stay out of it.
            if (!window[IP+'_active']) return;
            const now = performance.now();
            // Never start two clicks closer together than the holdoff
            if (now - clickDebounceTime < FLAP_CLICK_HOLDOFF_MS) {
              return;
            }
            clickDebounceTime = now;
            const view = getViewType();
            // Only play click in cockpit
            if (view === "cockpit") {
              playClick();
            } else {
            }
            // Trigger hum movement (existing logic will handle this)
            // This ensures hum continues to work as before
          }
          // Setup command listeners
          setupFlapCommandListener();
          // Poll for flap changes (hum only, no click)
          setInterval(async ()=>{
            try {
              const inst = geofs.aircraft.instance;
              if (!inst) return;
              // Find flap value
              let fl = null;
              const flapProps = {
                flapsPosition: inst.flapsPosition,
                flaps: inst.flaps,
                flapsTarget: inst.flapsTarget,
                'animation.values.flapsPosition': geofs?.animation?.values?.flapsPosition,
                'animation.values.flapsTarget': geofs?.animation?.values?.flapsTarget
              };
              for (const [prop, value] of Object.entries(flapProps)) {
                if (typeof value === "number" && !isNaN(value)) {
                  fl = value;
                  break;
                }
              }
              if (fl === null) return;
              await loadBuffers();
              if (lastFlap === null){
                lastFlap = fl;
                return;
              }
              const changeAmount = Math.abs(fl - lastFlap);
              // Detect aircraft ID
const ac = geofs.aircraft.instance.id;
// Set aircraft-specific flap sensitivity
let flapSensitivity = 0.002; // default fallback
if (ac === 24) {
    // A350-900 (slow flap animation)
    flapSensitivity = 0.012;   // or 0.015 if you want even smoother
}
else if (ac === 2973) {
    // A350-1000 (faster flap animation)
    flapSensitivity = 0.0005;
}
// Uses flapsTarget vs. current position (not a frame-to-frame delta) so low FPS doesn't cause flicker.
const flapTargetVal = (typeof inst.flapsTarget === "number" && !isNaN(inst.flapsTarget))
  ? inst.flapsTarget
  : (typeof geofs?.animation?.values?.flapsTarget === "number" && !isNaN(geofs.animation.values.flapsTarget) ? geofs.animation.values.flapsTarget : null);
const isMoving = flapTargetVal !== null
  ? Math.abs(flapTargetVal - fl) > flapSensitivity
  : changeAmount > flapSensitivity;
const now = performance.now();
// Use aircraft-specific threshold
if (isMoving) {
    lastMovementTime = now;
    const view = getViewType();
                // NO CLICK HERE - click only plays on commands
                // Cockpit: Low volume hum during movement
                if (view === "cockpit") {
                  if (!humSrc) {
                    startHum(0.60); // Increased from 0.45 (flap hum louder)
                  }
                }
                // Wing2: Loud hum only, NO clicking
                if (view === "wing2") {
                  if (!humSrc) {
                    startHum(0.80); // Increased from 0.6 (flap hum louder)
                  }
                }
                // WingEngine (WingL1/R1): closest to flaps, loudest hum
                if (view === "wingEngine") {
                  if (!humSrc) {
                    startHum(0.95);
                  }
                }
              } else if (humSrc && (now - lastMovementTime) > 500) {
                // Only stops once movement has been absent for a full 500ms, re-checked fresh each poll, to avoid stray-timeout races.
                stopHum();
              }
              lastFlap = fl;
            } catch(e){
              console.warn("[GE90 Flap] Poll error", e);
            }
          }, 100); // Check every 100ms for responsive detection
          // Test functions
          window['testFlapBuffers_'+SFX] = async function() {
            await loadBuffers();
            if (clickBuf) playClick();
            if (humBuf) startHum();
            setTimeout(() => stopHum(), 3000);
          };
          // Gear test function
          window['testGearSound_'+SFX] = async function() {
            const view = getViewType();
            // Test gear sound in current view
            await playGearMovement(view);
          };
// -------------------------------------------------------------
// SPOILER CLICK SOUND (cockpit + jumpseat only, dual-edge logic)
// -------------------------------------------------------------
(function setupSpoilerClick(){
    const ctx = window[IP+'_audioCtx'];
    const master = window[IP+'_master'];
    if (!ctx || !master) return;
    let spoilerBuf = null;
    let lastState = 0; // 0 = retracted, 1 = mid, 2 = fully deployed
    let debounce = 0;
    async function loadSpoilerBuffer(){
        if (spoilerBuf) return spoilerBuf;
        try {
            const r = await fetch(B.assets.spoiler, { mode: "cors" });
            const ab = await r.arrayBuffer();
            spoilerBuf = await ctx.decodeAudioData(ab);
        } catch(e){
            console.warn("[GE90 Spoiler] load failed", e);
        }
        return spoilerBuf;
    }
    function playSpoilerClick(){
        if (!spoilerBuf) return;
        const view = (typeof getViewType === "function")
            ? getViewType()
            : (window[IP+'_viewType'] || "exterior");
        if (view !== "cockpit" && view !== "jumpseat") return;
        const src = ctx.createBufferSource();
        src.buffer = spoilerBuf;
        src._GE90_TAG = true;
        const g = ctx.createGain();
        g.gain.value = 0.75; // Increased from 0.55 (spoiler click louder)
        g._GE90_TAG = true;
        src.connect(g);
        g.connect(master);
        src.start();
    }
setInterval(async () => {
    const now = performance.now();
    const val = geofs?.animation?.values?.airbrakesPosition;
    if (typeof val !== "number") return;
    // Two meaningful states:
    // 0 = retracted (0 → 0.05)
    // 1 = deployed (0.95 → 1)
    let currentState;
    if (val > 0.05) {
        currentState = 1; // spoilers moving or deployed
    } else {
        currentState = 0; // spoilers retracted or nearly retracted
    }
    // Only click when crossing the FIRST threshold of movement
    if (currentState !== lastState && (now - debounce) > 200) {
        debounce = now;
        await loadSpoilerBuffer();
        playSpoilerClick();
    }
    lastState = currentState;
}, 80);
})();
// Camera proximity effect: volume/tone/pitch shift by camera-to-aircraft distance, chase/free cam only.
(function setupProximityEffect(){
    const ctx = window[IP+'_audioCtx'];
    if (!ctx) return;
    // Tunables
    const POLL_MS           = 100;
    const SMOOTH_ALPHA_APPROACH = 0.35; // camera getting closer - same responsiveness as before
    const SMOOTH_ALPHA_RECEDE   = 0.75; // camera moving away - track faster so volume/tone fall off sooner
    // Default/starting distance (follow cam default) - prevents quiet startup
    const DEFAULT_DIST      = 64;
    // Distance range to map (meters) - DIST_NEAR set so default (~64m) sits at the
    // "normal" plateau (t=0), avoiding the "need to zoom in for normal volume" issue
    const DIST_NEAR         = 160; // widened from 70 so full volume/pitch is reached at a greater distance - ramp starts sooner and peaks sooner in chase cam
    const DIST_FAR          = 800;
    // Volume multiplier range
    const VOL_NEAR          = 1.05;
    const VOL_FAR           = 0.04;
    // Low-pass filter frequency range
    const LPF_NEAR_HZ       = 20000;
    const LPF_FAR_HZ        = 500;
    // Pitch multiplier range (strong distance-based pitch shift)
    const PITCH_NEAR        = 1.00;
    const PITCH_FAR         = 0.82;
    // Use logarithmic distance mapping (set true if zoom feels exponential)
    const USE_LOG_MAPPING   = true;
    let smoothedDist = DEFAULT_DIST;
    let lastMode = null;
    window[IP+'_proximityVolMult'] = 1.0;
    window[IP+'_proximityLpfHz'] = LPF_NEAR_HZ;
    window[IP+'_proximityPitchMult'] = 1.0;
    function clamp01(v){ return Math.max(0, Math.min(1, v)); }
    // -----------------------------------------------------------
    // WGS84 geodetic -> ECEF conversion
    // -----------------------------------------------------------
    function llaToEcef(lat, lon, alt){
        const a = 6378137;
        const e2 = 0.00669437999014;
        const latR = lat * Math.PI/180;
        const lonR = lon * Math.PI/180;
        const N = a / Math.sqrt(1 - e2 * Math.sin(latR)**2);
        const x = (N + alt) * Math.cos(latR) * Math.cos(lonR);
        const y = (N + alt) * Math.cos(latR) * Math.sin(lonR);
        const z = (N * (1-e2) + alt) * Math.sin(latR);
        return { x, y, z };
    }
    // -----------------------------------------------------------
    // Unified distance: camera (Cesium ECEF) to aircraft (LLA -> ECEF)
    // -----------------------------------------------------------
    function readCameraDistance(){
        try {
            const camPos = window.geofs?.camera?.cam?.position;
            const lla = window.geofs?.aircraft?.instance?.llaLocation;
            if (!camPos || !lla) return null;
            const acEcef = llaToEcef(lla[0], lla[1], lla[2]);
            const dx = camPos.x - acEcef.x;
            const dy = camPos.y - acEcef.y;
            const dz = camPos.z - acEcef.z;
            const dist = Math.sqrt(dx*dx + dy*dy + dz*dz);
            return isFinite(dist) ? dist : null;
        } catch(e){
            return null;
        }
    }
    // -----------------------------------------------------------
    // Wire proximity gain + LPF per layer. Retry-safe for async load.
    // Chain becomes: layer.filter -> pGain -> pLpf -> master
    // -----------------------------------------------------------
    function ensureProximityNodes(){
        if (!window[IP+'_proximityNodes']) window[IP+'_proximityNodes'] = {};
        const nodes = window[IP+'_proximityNodes'];
        ['idle','n1','toga','buzzsaw'].forEach(k => {
            if (nodes[k]) {
                // Hand off to the acoustic chain now that it's ready, if this layer initially fell back to master.
                if (!nodes[k].wiredToChain && window[IP+'_acousticChainInput']) {
                    try { nodes[k].lpf.disconnect(); } catch(e){}
                    try {
                        nodes[k].lpf.connect(window[IP+'_acousticChainInput']);
                        nodes[k].wiredToChain = true;
                    } catch(e){
                        try { nodes[k].lpf.connect(window[IP+'_master']); } catch(e2){}
                    }
                }
                return;
            }
            const layer = window[NP+'_layers']?.[k];
            if (!layer || !layer.gainNode || !layer.filter) return;
            try {
                const pGain = ctx.createGain();
                pGain.gain.value = 1.0;
                pGain._GE90_TAG = true;
                const pLpf = ctx.createBiquadFilter();
                pLpf.type = 'lowpass';
                pLpf.frequency.value = LPF_NEAR_HZ;
                pLpf.Q.value = 0.7;
                pLpf._GE90_TAG = true;
               try { layer.filter.disconnect(); } catch(e){}
                layer.filter.connect(pGain);
                pGain.connect(pLpf);
                // Route into acoustic chain if available, otherwise direct to
                // master for now — ensureProximityNodes() re-checks every poll
                // and hands off to the chain the moment it becomes ready.
                const chainReady = !!window[IP+'_acousticChainInput'];
                const chainInput = window[IP+'_acousticChainInput'] || window[IP+'_master'];
                pLpf.connect(chainInput);
                nodes[k] = { gain: pGain, lpf: pLpf, wiredToChain: chainReady };
                layer._GE90_proximityOwned = true;
            } catch(e){
                console.warn("[GE90 Proximity] wiring failed for", k, e);
            }
        });
        return nodes;
    }
    function applyProximity(){
        try {
            const nodes = ensureProximityNodes();
            const targetHz = window[IP+'_proximityLpfHz'] || LPF_NEAR_HZ;
            const targetVol = window[IP+'_proximityVolMult'] || 1.0;
            Object.values(nodes).forEach(n => {
                if (n.lpf?.frequency) n.lpf.frequency.setTargetAtTime(targetHz, ctx.currentTime, 0.15);
                if (n.gain?.gain) n.gain.gain.setTargetAtTime(targetVol, ctx.currentTime, 0.15);
            });
        } catch(e){}
    }
    setInterval(() => {
        try {
            const mode = window.geofs?.camera?.currentModeName;
            if (mode !== lastMode) {
                lastMode = mode;
                smoothedDist = DEFAULT_DIST;
            }
            // Only apply proximity effect in chase/free - follow stays neutral
            if (mode !== 'chase' && mode !== 'free') {
                window[IP+'_proximityVolMult'] = 1.0;
                window[IP+'_proximityLpfHz'] = LPF_NEAR_HZ;
                window[IP+'_proximityPitchMult'] = 1.0;
                applyProximity();
                return;
            }
            const rawDist = readCameraDistance();
            if (typeof rawDist !== 'number' || !isFinite(rawDist) || rawDist <= 0) {
                // Hold last known values instead of snapping to neutral
                applyProximity();
                return;
            }
            const _geProxDelta = rawDist - smoothedDist;
            const _geProxAlpha = (_geProxDelta > 0) ? SMOOTH_ALPHA_RECEDE : SMOOTH_ALPHA_APPROACH;
            smoothedDist += _geProxDelta * _geProxAlpha;
            let t;
            if (USE_LOG_MAPPING) {
                const logNear = Math.log(Math.max(1, DIST_NEAR));
                const logFar = Math.log(Math.max(2, DIST_FAR));
                const logDist = Math.log(Math.max(1, smoothedDist));
                t = clamp01((logDist - logNear) / (logFar - logNear));
            } else {
                t = clamp01((smoothedDist - DIST_NEAR) / (DIST_FAR - DIST_NEAR));
            }
            const volMult = VOL_NEAR + (VOL_FAR - VOL_NEAR) * t;
            const lpfHz = LPF_NEAR_HZ + (LPF_FAR_HZ - LPF_NEAR_HZ) * t;
            const pitchMult = PITCH_NEAR + (PITCH_FAR - PITCH_NEAR) * t;
            window[IP+'_proximityVolMult'] = volMult;
            window[IP+'_proximityLpfHz'] = lpfHz;
            window[IP+'_proximityPitchMult'] = pitchMult;
            applyProximity();
        } catch(e){
            // On error, hold last values rather than reset
            applyProximity();
        }
    }, POLL_MS);
})();
        })();
        // Persistent mute state
        window[IP+'_persistentlyMuted'] = false;
        // Force mute all HTML5 audio elements
        function forceMuteAllAudio() {
          const audioElements = document.querySelectorAll('audio');
          audioElements.forEach(audio => {
            audio.muted = true;
            audio.volume = 0;
            audio._GE90_FORCED_MUTE_BY_PACK = true;
            // Override play method to prevent audio from playing
            if (!audio._geofsMutePatched) {
              const originalPlay = audio.play;
              audio.play = () => {
                if (window[IP+'_persistentlyMuted']) {
                  return Promise.resolve();
                }
                audio.muted = false;
                audio.volume = 1;
                audio._GE90_FORCED_MUTE_BY_PACK = false;
                return originalPlay.call(audio);
              };
              audio._geofsMutePatched = true;
            }
          });
          // Also force mute GeoFS sounds
          if (typeof geofs !== 'undefined' && geofs.sound && geofs.sound.sounds) {
            Object.keys(geofs.sound.sounds).forEach(key => {
              if (geofs.sound.sounds[key]) {
                geofs.sound.sounds[key].volume = 0;
                geofs.sound.sounds[key].muted = true;
              }
            });
          }
        }
        // Show mute indicator
        function showMuteIndicator() {
          const existing = document.getElementById('geofs-mute-indicator');
          if (existing) existing.remove();
          const indicator = document.createElement('div');
          indicator.id = 'geofs-mute-indicator';
          indicator.textContent = window[IP+'_persistentlyMuted'] ? '🔇 PERSISTENT MUTE' : '🔊 UNMUTED';
          indicator.style.cssText = `
            position: fixed;
            top: 20px;
            right: 20px;
            background: ${window[IP+'_persistentlyMuted'] ? 'rgba(220, 53, 69, 0.9)' : 'rgba(40, 167, 69, 0.9)'};
            color: white;
            padding: 12px 20px;
            border-radius: 8px;
            font-size: 14px;
            font-weight: bold;
            z-index: 10000;
            transition: opacity 0.3s;
            box-shadow: 0 4px 12px rgba(0,0,0,0.3);
            font-family: Arial, sans-serif;
          `;
          document.body.appendChild(indicator);
          setTimeout(() => {
            if (indicator.parentNode) {
              indicator.style.opacity = '0';
              setTimeout(() => {
                if (indicator.parentNode) {
                  indicator.parentNode.removeChild(indicator);
                }
              }, 300);
            }
          }, 3000);
        }
        // Monitor for new audio elements
        function monitorAudioElements() {
          const observer = new MutationObserver((mutations) => {
            mutations.forEach((mutation) => {
              mutation.addedNodes.forEach((node) => {
                if (node.tagName === 'AUDIO') {
                  if (window[IP+'_persistentlyMuted']) {
                    node.muted = true;
                    node.volume = 0;
                    node._GE90_FORCED_MUTE_BY_PACK = true;
                  }
                }
                if (node.querySelectorAll) {
                  const audioElements = node.querySelectorAll('audio');
                  audioElements.forEach(audio => {
                    if (window[IP+'_persistentlyMuted']) {
                      audio.muted = true;
                      audio.volume = 0;
                      audio._GE90_FORCED_MUTE_BY_PACK = true;
                    }
                  });
                }
              });
            });
          });
          observer.observe(document.body, {
            childList: true,
            subtree: true
          });
        }
        // Start audio monitoring
        monitorAudioElements();
        // Continuous monitoring to ensure mute stays active
        let muteMonitoringInterval = null;
        function startContinuousMuteMonitoring() {
          if (muteMonitoringInterval) return;
          muteMonitoringInterval = setInterval(() => {
            if (window[IP+'_persistentlyMuted']) {
              forceMuteAllAudio();
            } else {
              clearInterval(muteMonitoringInterval);
              muteMonitoringInterval = null;
            }
          }, 100);
        }
        // Test function to identify sliders (for debugging)
        function debugSliders() {
            const sliders = document.querySelectorAll(".slider-selection");
            sliders.forEach((s, i) => {
                let parent = s.closest(".geofs-preference") || s.parentElement;
            });
        }
        // Volume UI now lives in the single shared GE-Ultimate slider. View-dependent lowpass on idle/n1/toga/buzzsaw: muffled in cockpit/wing2, raw in wingEngine/exterior.
        const _ENGINE_LPF_DEFAULT = B.engineLpf.default;
        const _ENGINE_LPF_INTERIOR = B.engineLpf.interior;
        function setEngineLayerLPF(viewType){
          const layers = window[NP+'_layers'];
          if (!layers) return;
          const interiorMap = _ENGINE_LPF_INTERIOR[viewType] || null;
          Object.keys(_ENGINE_LPF_DEFAULT).forEach((k) => {
            const layer = layers[k];
            if (!layer?.filter?.frequency) return;
            const interiorTarget = interiorMap ? interiorMap[k] : null;
            const target = (interiorTarget != null)
              ? Math.min(interiorTarget, _ENGINE_LPF_DEFAULT[k])
              : _ENGINE_LPF_DEFAULT[k];
            layer.filter.frequency.setTargetAtTime(target, ctx.currentTime, 0.15);
          });
        }
        // -------------------------
        // Effective mute + attenuation + interior/exterior blend + GeoFS volume sync
        // -------------------------
        function applyEffectiveMute(){
          try {
            window[IP+'_siteMuted'] = detectSiteMuted();
            const effectiveMuted =
              !!window[IP+'_userMuted'] ||
              !!window[IP+'_siteMuted'] ||
              !!window[IP+'_paused'] ||
              !!window[IP+'_persistentlyMuted'];  // Add persistent mute
            window[IP+'_muted'] = effectiveMuted;
            // Force mute all HTML5 audio elements if persistently muted
            if (window[IP+'_persistentlyMuted']) {
              forceMuteAllAudio();
            }
            // Volume is now handled by the single shared GE-Ultimate volume slider (see bottom of file)
            const viewType = getViewType();
            window[IP+'_viewType'] = viewType;
            setEngineLayerLPF(viewType);
            const exteriorBoost = window[IP+'_master_boost'] || B.mix.masterBoost;
         let target;
            if (!window[IP+'_active']) {
              target = 0; // aircraft gating — pack inactive for this aircraft
            } else if (effectiveMuted) {
              target = 0;
          } else if (viewType === 'wing2') {
              target = exteriorBoost * B.mix.wing2Atten;
            } else if (viewType === 'wingEngine') {
              target = exteriorBoost * 0.50;
            } else {
              target = exteriorBoost;
            }
            master.gain.cancelScheduledValues(ctx.currentTime);
            master.gain.setTargetAtTime(
              Math.max(0, Math.min(6, target)),
              ctx.currentTime,
              0.02
            );
            // Interior/exterior crossfade
            let interiorBlend = B.mix.exteriorInteriorBlend;
            if (!effectiveMuted) {
              if (viewType === "cockpit") interiorBlend = B.mix.cockpitInteriorBlend;
              else if (viewType === "wing2") interiorBlend = B.mix.wing2InteriorBlend;
            }
            const exteriorBlend = 1 - interiorBlend;
            extGain.gain.setTargetAtTime(exteriorBlend, ctx.currentTime, 0.15);
            intGain.gain.setTargetAtTime(interiorBlend, ctx.currentTime, 0.15);
            if (viewType === "cockpit") {
  intLPF.frequency.setTargetAtTime(2400, ctx.currentTime, 0.15); // more bass emphasis
  intLowShelf.gain.setTargetAtTime(6.0, ctx.currentTime, 0.15);   // stronger low boost
  intHighShelf.gain.setTargetAtTime(-1.0, ctx.currentTime, 0.15); // reduce harsh highs
  // Slightly reduce final master in cockpit to avoid proximity overload —
  // but never undo an active mute/pause (this previously ran unconditionally
  // and silently un-muted engine audio while muted in this view).
  if (!effectiveMuted) {
    master.gain.setTargetAtTime(Math.max(0, Math.min(6, (window[IP+'_master_boost'] || B.mix.masterBoost) * B.mix.cockpitAtten * 0.95)), ctx.currentTime, 0.02);
  }
}
else if (viewType === "wing2") {
  intLPF.frequency.setTargetAtTime(2600, ctx.currentTime, 0.15);
  intLowShelf.gain.setTargetAtTime(6.0, ctx.currentTime, 0.15);
  intHighShelf.gain.setTargetAtTime(0.0, ctx.currentTime, 0.15);
  if (!effectiveMuted) {
    master.gain.setTargetAtTime(Math.max(0, Math.min(6, (window[IP+'_master_boost'] || B.mix.masterBoost) * B.mix.wing2Atten * 0.98)), ctx.currentTime, 0.02);
  }
}
else {
  intLPF.frequency.setTargetAtTime(20000, ctx.currentTime, 0.15);
  intLowShelf.gain.setTargetAtTime(0.0, ctx.currentTime, 0.15);
  intHighShelf.gain.setTargetAtTime(0.0, ctx.currentTime, 0.15);
}
            intSmooth.gain.setTargetAtTime(1, ctx.currentTime, B.mix.interiorSmoothTc);
            if (viewType === "wing2" && !effectiveMuted) {
              lfoGain.gain.setTargetAtTime(B.mix.wing2VibeDepth, ctx.currentTime, 0.25);
              intVibe.gain.setTargetAtTime(1, ctx.currentTime, 0.25);
            } else {
              lfoGain.gain.setTargetAtTime(0, ctx.currentTime, 0.25);
              intVibe.gain.setTargetAtTime(1, ctx.currentTime, 0.25);
            }
            // Ambience volume control
            try {
              const ambCA = window[IP+'_ambienceCockpitA'];
              const ambCB = window[IP+'_ambienceCockpitB'];
              const ambW  = window[IP+'_ambienceWing2'];
              const ambE  = window[IP+'_ambienceExterior'];
             const ambWAir = window[IP+'_ambienceWing2Air'];
             const ambWAirB = window[IP+'_ambienceWing2AirB'];
              let cockpitTarget = 0;
              let wing2GroundTarget = 0;
              let wing2AirTarget = 0;
              let exteriorTarget = 0;
              // Authoritative ground/air detection (same source as rattle system)
              const isGrounded = !!(window.geofs?.aircraft?.instance?.groundContact);
              if (!effectiveMuted) {
                if (viewType === "cockpit") {
                  // Per-aircraft extra cabin-ambience boost for cockpit view
                  // (defaults to 1 = no change for any pack that doesn't set it).
                  const ambBoostCockpit = B.mix.ambBoostCockpit || 1;
                  cockpitTarget = 1.95 * ambBoostCockpit; // 1.50 x1.30 (30% cabin ambience boost)
               } else if (viewType === "wing2" || viewType === "wingEngine") {
                  // Per-aircraft extra cabin-ambience boost for these two views only
                  // (defaults to 1 = no change for any pack that doesn't set it).
                  const ambBoost = B.mix.ambBoostWing || 1;
                  if (isGrounded) {
                    // wingEngine is closer to engines so slightly louder ground ambience
                    wing2GroundTarget = ((viewType === "wingEngine") ? 0.91 : 1.04) * ambBoost; // x1.30 (30% cabin ambience boost)
                  } else {
                    // Speed-based airborne ambience volume
                    // Scales from 0.30 (liftoff) to 0.80 (Mach 0.89 / ~300 m/s)
                    const tas = window.geofs?.aircraft?.instance?.trueAirSpeed || 0;
                    const TAS_MIN = 0;
                    const TAS_MAX = 300;
                    const VOL_AIR_MIN = 0.364 * ambBoost; // 0.28 x1.30 (30% cabin ambience boost)
                    const VOL_AIR_MAX = ((viewType === "wingEngine") ? 0.91 : 1.04) * ambBoost; // x1.30 (30% cabin ambience boost)
                    const tSpeed = Math.max(0, Math.min(1, (tas - TAS_MIN) / (TAS_MAX - TAS_MIN)));
                    wing2AirTarget = VOL_AIR_MIN + (VOL_AIR_MAX - VOL_AIR_MIN) * tSpeed;
                  }
                }else if (viewType === "exterior") {
                  exteriorTarget = B.mix.exteriorAmbLevel;
                }
              }
              if (ambCA) ambCA.ambGain.gain.setTargetAtTime(cockpitTarget * 0.6, ctx.currentTime, 0.35);
              if (ambCB) ambCB.ambGain.gain.setTargetAtTime(cockpitTarget * 0.4, ctx.currentTime, 0.35);
              if (ambW)  ambW.ambGain.gain.setTargetAtTime(wing2GroundTarget, ctx.currentTime, 0.35);
              if (ambWAir) ambWAir.ambGain.gain.setTargetAtTime(wing2AirTarget * 0.6, ctx.currentTime, 0.35);
              if (ambWAirB) ambWAirB.ambGain.gain.setTargetAtTime(wing2AirTarget * 0.4, ctx.currentTime, 0.35);
              if (ambE)  ambE.ambGain.gain.setTargetAtTime(exteriorTarget, ctx.currentTime, 0.35);
              // Hard-zero exterior ambience in non-exterior views
              if (viewType !== "exterior" && ambE) {
                ambE.ambGain.gain.setValueAtTime(0, ctx.currentTime);
              }
            } catch(e){}
            // If view becomes exterior while gear sound is active, fade it out
            if (viewType === "exterior") {
              stopGearSoundSmooth();
            }
          } catch(e){}
        }
       // Keyboard S/P (mute/pause): the outer listener attached before start() delegates here once these exist.
        function toggleMute(){
          window[IP+'_userMuted'] = !window[IP+'_userMuted'];
          window[IP+'_persistentlyMuted'] = window[IP+'_userMuted'];
          applyEffectiveMute();
          showMuteIndicator();
          startContinuousMuteMonitoring();
          // Native GeoFS audio mute is handled entirely by the shared
          // cross-pack arbiter — not touched here, to avoid one pack's
          // mute toggle affecting another active pack's native audio state.
        }
        function togglePause(){
          window[IP+'_paused'] = !window[IP+'_paused'];
          window[IP+'_userPaused'] = window[IP+'_paused'];
          if (window[IP+'_paused']) {
            try { ctx.suspend(); } catch(e){}
          } else {
            try { ctx.resume(); } catch(e){}
          }
          applyEffectiveMute();
        }
        window[IP+'_toggleMuteFull']  = toggleMute;
        window[IP+'_togglePauseFull'] = togglePause;
        window[IP+'_applyEffectiveMute'] = applyEffectiveMute;
        // -------------------------
        // Periodic view/mute updates + smooth volume sync
        // -------------------------
        setInterval(applyEffectiveMute, 250);
        // Sync the paused global with geofs.pause after map spawns (GeoFS
        // unpausing itself, e.g. a right-click spawn, resumes our context too).
        (function watchGeofsPauseState(){
            let lastGeofsPause = null;
            setInterval(() => {
                try {
                    const geofsPaused = !!window.geofs?.pause;
                    if (lastGeofsPause === null) {
                        lastGeofsPause = geofsPaused;
                        return;
                    }
                    // GeoFS just unpaused itself (map spawn while we were paused)
                    if (lastGeofsPause === true && geofsPaused === false) {
                        if (window[IP+'_paused'] && !window[IP+'_userPaused']) {
                            window[IP+'_paused'] = false;
                            window[IP+'_userPaused'] = false;
                            window[IP+'_persistentlyMuted'] = window[IP+'_userMuted'] || false;
                            try { ctx.resume(); } catch(e){}
                            applyEffectiveMute();
                        }
                    }
                    lastGeofsPause = geofsPaused;
                } catch(e){}
            }, 150);
        })();
        // Right-click map spawn safety net — resume ctx if it was suspended
        document.addEventListener('contextmenu', () => {
            setTimeout(() => {
                try {
                    if (!window.geofs?.pause && window[IP+'_paused'] && !window[IP+'_userPaused']) {
                        window[IP+'_paused'] = false;
                            window[IP+'_userPaused'] = false;
                        window[IP+'_persistentlyMuted'] = window[IP+'_userMuted'] || false;
                        try { ctx.resume(); } catch(e){}
                        applyEffectiveMute();
                    }
                } catch(e){}
            }, 800);
        });
        // Note: the shared GE-Ultimate volume slider's smoothing loop (bottom of file) handles this now.
        // Test slider identification (call manually to debug)
        window['debugGeoFSSliders_'+SFX] = debugSliders;
        // Emergency functions
        window['emergencyMute_'+SFX] = function() {
          window[IP+'_persistentlyMuted'] = true;
          window[IP+'_userMuted'] = true;
          forceMuteAllAudio();
          showMuteIndicator();
          startContinuousMuteMonitoring();
        };
        window['emergencyUnmute_'+SFX] = function() {
          window[IP+'_persistentlyMuted'] = false;
          window[IP+'_userMuted'] = false;
          showMuteIndicator();
          if (muteMonitoringInterval) {
            clearInterval(muteMonitoringInterval);
            muteMonitoringInterval = null;
          }
        };
      } catch(e){
        console.warn("[GE90] setupPageContext failed", e);
      }
    });
  })();
  })();
  // =====================================================================
  }
  // =====================================================================
  // BANKS — one declarative description per aircraft. Everything the
  // runtime needs to sound like this specific type lives here.
  // =====================================================================
  const GEFMOD_BANKS = [
    // ---- A320 CEO Family -----------------------------------------------
    {
      "code": "a320",
      "suffix": "A320",
      "name": "A320 CEO Family",
      "ids": ["5156", "2879", "3534", "3011", "5086", "2870"],
      "layers": {
        "idle": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Airbus-A320-Sound-Pack/main/A320IDLE.mp3",
        "n1": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Airbus-A320-Sound-Pack/main/A320N1.mp3",
        "toga": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A320-CEO-Family-Sound-Repository/main/A320TOGA3.mp3"
      },
      "layerLpf": {
        "idle": 6000,
        "n1": 6000,
        "toga": 1000
      },
      "engineLpf": {
        "default": {
          "idle": 6000,
          "n1": 6000,
          "toga": 1000,
          "buzzsaw": 3200
        },
        "interior": {
          "cockpit": {
            "idle": 1500,
            "n1": 1500,
            "toga": 800,
            "buzzsaw": 1500
          },
          "wing2": {
            "idle": 4200,
            "n1": 4200,
            "toga": 600,
            "buzzsaw": 1700
          },
          "wingEngine": {
            "idle": 6000,
            "n1": 6000,
            "toga": 1000,
            "buzzsaw": 3200
          }
        }
      },
      "profiles": {
        "wingEngine": {
          "gain": 0.65,
          "lowShelfFreq": 275,
          "lowShelfGain": 6,
          "peakFreq": 1000,
          "peakGain": -4,
          "peakQ": 1.3,
          "highShelfFreq": 2000,
          "highShelfGain": -6,
          "lpfFreq": 3800
        },
        "wing2": {
          "gain": 0.55,
          "lowShelfFreq": 300,
          "lowShelfGain": 6,
          "peakFreq": 800,
          "peakGain": -5,
          "peakQ": 0.8,
          "highShelfFreq": 1800,
          "highShelfGain": -8,
          "lpfFreq": 3000
        },
        "cockpit": {
          "gain": 1,
          "lowShelfFreq": 200,
          "lowShelfGain": 0,
          "peakFreq": 1000,
          "peakGain": 0,
          "peakQ": 1,
          "highShelfFreq": 3000,
          "highShelfGain": 0,
          "lpfFreq": 20000
        },
        "exterior": {
          "gain": 1,
          "lowShelfFreq": 200,
          "lowShelfGain": 0,
          "peakFreq": 1000,
          "peakGain": 0,
          "peakQ": 1,
          "highShelfFreq": 3000,
          "highShelfGain": 0,
          "lpfFreq": 20000
        },
        "exteriorFront": {
          "gain": 1,
          "lowShelfFreq": 250,
          "lowShelfGain": -3,
          "peakFreq": 1400,
          "peakGain": 2,
          "peakQ": 1.1,
          "highShelfFreq": 2200,
          "highShelfGain": 3,
          "lpfFreq": 16000
        },
        "exteriorBehind": {
          "gain": 1,
          "lowShelfFreq": 250,
          "lowShelfGain": 4,
          "peakFreq": 900,
          "peakGain": -1.5,
          "peakQ": 0.9,
          "highShelfFreq": 2200,
          "highShelfGain": -6,
          "lpfFreq": 7000
        }
      },
      "views": {
        "cockpit": ["cockpit", "jumpseat", "jump seat", "virtual cockpit", "vc"],
        "wing2": ["wing 2", "wing2", "left wing", "right wing"],
        "side": ["left wing", "right wing", "right", "left"],
        "engine": ["cfm", "iaev", "v2500", "engine"],
        "hasWingEngine": true
      },
      "views2": {
        "cockpit": ["cockpit", "jumpseat", "jump seat", "virtual cockpit", "vc", "2d cockpit"],
        "wing2": ["wing 2", "wing2", "left wing", "right wing", "wing view", "external wing"],
        "side": [],
        "engine": [],
        "hasWingEngine": false
      },
      "mix": {
        "masterBoost": 1.85,
        "cockpitAtten": 0.5,
        "wing2Atten": 0.6,
        "preGainDb": 0,
        "compThresholdDb": -15,
        "compRatio": 1,
        "cockpitInteriorBlend": 0.95,
        "wing2InteriorBlend": 0.9,
        "exteriorInteriorBlend": 0,
        "wing2EngineAtten": 0.55,
        "wingEngineAtten": 0.65,
        "interiorSmoothTc": 0.025,
        "wing2VibeDepth": 0.015,
        "wing2VibeRate": 2.5,
        "exteriorAmbLevel": 0.2,
        "exteriorLpfFreq": 1800,
        "tireScreechVol": 0.4,
        "togaGain": 1.15,
        "engineLayerAtten": 1,
        "idleWing2Boost": 1
      },
      "engine": {
        "rpmMin": 1000,
        "rpmShutdown": 990,
        "startupSuppressMs": 2000,
        "spawnProtectionMs": 8000,
        "altitudeProtectionFt": 1000,
        "idleUnfiltered": false
      },
      "curve": {
        "idleCut": 0.45,
        "n1Start": 0.12,
        "n1Full": 0.52,
        "togaStart": 0.6,
        "midCenter": 0.33,
        "midWidth": 0.32,
        "midGain": 2,
        "floor": 0.1,
        "togaExp": 1.25,
        "bellExp": 1.6,
        "idleBell": 0.65,
        "n1Bell": "bell",
        "togaPow": 1.1,
        "idleFloorMul": 1.25,
        "idleFinalMul": 1,
        "n1FloorMul": 0.95,
        "togaBoost": 1.2,
        "togaFinalMul": null,
        "idleRateMul": 9,
        "idleRateTail": 0.9,
        "n1RateK": 0.22,
        "togaRateK": 0.2,
        "togaRateExp": 1.15,
        "idlePower": 0.5,
        "idlePitchIntensity": 0.75,
        "idleVolumeBoost": 1.5,
        "n1Scale": 0.42,
        "maxLayerGain": 1.8,
        "n1BellMode": "bell",
        "n1BellMul": 1
      },
      "buzzsaw": {
        "lpf": 3200,
        "altMaxFt": 8000,
        "altExp": 1.6,
        "altScale": 1.5,
        "throttleStart": 0.65,
        "throttleEnd": 1,
        "rateBase": 0.78,
        "rateSpan": 0.2,
        "wingEngineBoost": 1.5,
        "wing2Atten": 1,
        "cockpitAtten": 1
      },
      "toga": {
        "altVolume": null,
        "cockpitMult": 1,
        "altitudeHF": false
      },
      "gear": {
        "lpfFreq": 1400,
        "lpfQ": 0.6,
        "fadeIn": 0.4,
        "fadeOut": 0.4,
        "volCockpit": 0.633,
        "volWing2": 0.437,
        "volWingEngine": 0.748
      },
      "assets": {
        "ambWing2Air": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A320-CEO-Family-Sound-Pack-Official/main/freesound_community-airplane-interior-ambience-59644.mp3",
        "ambWing2": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/freesound_community-plane-interior-52034%20(1).mp3",
        "ambExterior": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Weather_%23.wav",
        "gearClick": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Master_9.wav",
        "spoiler": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-Sound-Pack-Official/main/Master_10.wav",
        "engineToggle": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Master_19.wav",
        "rain": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Airbus-A320-Sound-Pack/main/Weather_6.wav",
        "tire": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-Sound-Pack-Official/main/freesound_community-cinematic-deep-rumble-6418.mp3",
        "tireScreech": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Sound-Pack-Ultimate-/main/Tire%20Screech.wav",
        "rattle": {
          "low": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Dynamics_20.wav",
          "midL": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Dynamics_2.wav",
          "transC": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Dynamics_18.wav"
        },
        "startupExt": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Airbus-A320-Sound-Pack/main/A320ENGINESTART.wav",
        "startupInt": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Airbus-A320-Sound-Pack/main/A320%20star2_INN.wav",
        "shutdown": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Airbus-A320-Sound-Pack/main/A320%20Engine%20ShutOff.wav",
        "ambCockpit": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Airbus-A320-Sound-Pack/main/Master_24.wav",
        "buzzsaw": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A320-CEO-Family-Sound-Repository/main/A320Buzzsaw.wav",
        "gearHum": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-Sound-Pack-Official/main/Gear%20Down%20(3).wav",
        "tray": "",
        "flapClick": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A320NEO-Sound-Repository/main/AirbusFlapSound.wav",
        "flapMotor": null,
        "flapHum": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Airbus-A320-Sound-Pack/main/Dynamics_5.wav",
        "configAlarm": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Sound-Pack-Ultimate-/main/Airbus%20Alarm.wav"
      }
    },
    // ---- A320neo -------------------------------------------------------
    {
      "code": "a320neo",
      "suffix": "A320NEO",
      "name": "A320neo",
      "ids": ["5847", "2871", "2865", "242", "4646"],
      "layers": {
        "idle": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A320NEO-Sound-Repository/main/A320NEOIDLE.wav",
        "n1": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A320NEO-Sound-Repository/main/A320NEON1.wav",
        "toga": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A320NEO-Sound-Repository/main/A320NEOTOGA.wav"
      },
      "layerLpf": {
        "idle": 10000,
        "n1": 10000,
        "toga": 16000
      },
      "engineLpf": {
        "default": {
          "idle": 10000,
          "n1": 10000,
          "toga": 16000,
          "buzzsaw": 10000
        },
        "interior": {
          "cockpit": {
            "idle": 1500,
            "n1": 1500,
            "toga": 800,
            "buzzsaw": 1500
          },
          "wing2": {
            "idle": 4200,
            "n1": 4200,
            "toga": 900,
            "buzzsaw": 1700
          },
          "wingEngine": {
            "idle": 10000,
            "n1": 10000,
            "toga": 16000,
            "buzzsaw": 10000
          }
        }
      },
      "profiles": {
        "wingEngine": {
          "gain": 0.55,
          "lowShelfFreq": 275,
          "lowShelfGain": 6,
          "peakFreq": 800,
          "peakGain": -4.5,
          "peakQ": 1.2,
          "highShelfFreq": 1800,
          "highShelfGain": -7,
          "lpfFreq": 3400
        },
        "wing2": {
          "gain": 0.5,
          "lowShelfFreq": 300,
          "lowShelfGain": 6,
          "peakFreq": 600,
          "peakGain": -5,
          "peakQ": 0.8,
          "highShelfFreq": 1600,
          "highShelfGain": -8,
          "lpfFreq": 3000
        },
        "cockpit": {
          "gain": 1,
          "lowShelfFreq": 200,
          "lowShelfGain": 0,
          "peakFreq": 1000,
          "peakGain": 0,
          "peakQ": 1,
          "highShelfFreq": 3000,
          "highShelfGain": 0,
          "lpfFreq": 20000
        },
        "exterior": {
          "gain": 1,
          "lowShelfFreq": 200,
          "lowShelfGain": 0,
          "peakFreq": 1000,
          "peakGain": 0,
          "peakQ": 1,
          "highShelfFreq": 3000,
          "highShelfGain": 0,
          "lpfFreq": 20000
        },
        "exteriorFront": {
          "gain": 1,
          "lowShelfFreq": 250,
          "lowShelfGain": -3,
          "peakFreq": 1400,
          "peakGain": 2,
          "peakQ": 1.1,
          "highShelfFreq": 2200,
          "highShelfGain": 3,
          "lpfFreq": 16000
        },
        "exteriorBehind": {
          "gain": 1,
          "lowShelfFreq": 250,
          "lowShelfGain": 4,
          "peakFreq": 900,
          "peakGain": -1.5,
          "peakQ": 0.9,
          "highShelfFreq": 2200,
          "highShelfGain": -6,
          "lpfFreq": 7000
        }
      },
      "views": {
        "cockpit": ["cockpit", "jumpseat", "jump seat", "virtual cockpit", "vc"],
        "wing2": ["wing 2", "wing2", "left wing", "right wing"],
        "side": ["left wing", "right wing", "right", "left"],
        "engine": ["cfm", "iaev", "v2500", "engine"],
        "hasWingEngine": true
      },
      "views2": {
        "cockpit": ["cockpit", "jumpseat", "jump seat", "virtual cockpit", "vc", "2d cockpit"],
        "wing2": ["wing 2", "wing2", "left wing", "right wing", "wing view", "external wing"],
        "side": [],
        "engine": [],
        "hasWingEngine": false
      },
      "mix": {
        "masterBoost": 2.2,
        "cockpitAtten": 0.32,
        "wing2Atten": 0.55,
        "preGainDb": -2,
        "compThresholdDb": -8,
        "compRatio": 0.88,
        "cockpitInteriorBlend": 0.95,
        "wing2InteriorBlend": 0.9,
        "exteriorInteriorBlend": 0,
        "wing2EngineAtten": 0.55,
        "wingEngineAtten": 0.85,
        "interiorSmoothTc": 0.025,
        "wing2VibeDepth": 0.015,
        "wing2VibeRate": 2.5,
        "exteriorAmbLevel": 0.2,
        "exteriorLpfFreq": 1800,
        "tireScreechVol": 0.45,
        "togaGain": 1.15,
        "engineLayerAtten": 1.4,
        "idleWing2Boost": 1,
        "ambBoostWing": 1.5,
        "ambBoostCockpit": 1.5
      },
      "engine": {
        "rpmMin": 1000,
        "rpmShutdown": 990,
        "startupSuppressMs": 2000,
        "spawnProtectionMs": 8000,
        "altitudeProtectionFt": 1000,
        "idleUnfiltered": false
      },
      "curve": {
        "idleCut": 0.45,
        "n1Start": 0.12,
        "n1Full": 0.52,
        "togaStart": 0.42,
        "midCenter": 0.33,
        "midWidth": 0.32,
        "midGain": 2,
        "floor": 0.1,
        "togaExp": 1.25,
        "bellExp": 1.6,
        "idleBell": 0.65,
        "n1Bell": 0.75,
        "togaPow": 1.1,
        "idleFloorMul": 1.25,
        "idleFinalMul": 1,
        "n1FloorMul": 0.95,
        "togaBoost": 1,
        "togaFinalMul": null,
        "idleRateMul": 9,
        "idleRateTail": 0.9,
        "n1RateK": 0.22,
        "togaRateK": 0.5,
        "togaRateExp": 1.15,
        "idlePower": 0.38,
        "idlePitchIntensity": 0.28,
        "idleVolumeBoost": 0.5,
        "n1Scale": 0.35,
        "maxLayerGain": 0.9,
        "n1BellMode": "const",
        "n1BellMul": 0.75
      },
      "buzzsaw": {
        "lpf": 10000,
        "altMaxFt": 8000,
        "altExp": 1.6,
        "altScale": 1.5,
        "throttleStart": 0.65,
        "throttleEnd": 1,
        "rateBase": 0.78,
        "rateSpan": 0.2,
        "wingEngineBoost": 1,
        "wing2Atten": 1,
        "cockpitAtten": 1
      },
      "toga": {
        "altVolume": null,
        "cockpitMult": 1,
        "altitudeHF": false
      },
      "gear": {
        "lpfFreq": 1400,
        "lpfQ": 0.6,
        "fadeIn": 0.4,
        "fadeOut": 0.4,
        "volCockpit": 0.633,
        "volWing2": 0.437,
        "volWingEngine": 0.748
      },
      "assets": {
        "ambWing2Air": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A320-CEO-Family-Sound-Pack-Official/main/freesound_community-airplane-interior-ambience-59644.mp3",
        "ambWing2": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/freesound_community-plane-interior-52034%20(1).mp3",
        "ambExterior": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Weather_%23.wav",
        "gearClick": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Master_9.wav",
        "spoiler": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-Sound-Pack-Official/main/Master_10.wav",
        "engineToggle": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Master_19.wav",
        "rain": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Airbus-A320-Sound-Pack/main/Weather_6.wav",
        "tire": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-Sound-Pack-Official/main/freesound_community-cinematic-deep-rumble-6418.mp3",
        "tireScreech": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Sound-Pack-Ultimate-/main/Tire%20Screech.wav",
        "rattle": {
          "low": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Dynamics_20.wav",
          "midL": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Dynamics_2.wav",
          "transC": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Dynamics_18.wav"
        },
        "startupExt": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A320NEO-Sound-Repository/main/A320NEOSTARTUPSOUND.wav",
        "startupInt": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A320NEO-Sound-Repository/main/A320NEOINTERIORSTARTUP.wav",
        "shutdown": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A320NEO-Sound-Repository/main/A320NEOSHUTDOWN.wav",
        "ambCockpit": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B737NG-Family-Sound-Pack/main/B737CockpitAmbience.wav",
        "buzzsaw": "",
        "gearHum": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-Sound-Pack-Official/main/Gear%20Down%20(3).wav",
        "tray": "",
        "flapClick": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A320NEO-Sound-Repository/main/AirbusFlapSound.wav",
        "flapMotor": null,
        "flapHum": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Airbus-A320-Sound-Pack/main/Dynamics_5.wav",
        "configAlarm": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Sound-Pack-Ultimate-/main/Airbus%20Alarm.wav"
      }
    },
    // ---- A330 ----------------------------------------------------------
    {
      "code": "a330",
      "suffix": "A330",
      "name": "A330",
      "ids": ["244", "2856", "6012"],
      "layers": {
        "idle": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A330-Sound-Repository/blob/main/A330CEOIDLE.wav",
        "n1": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A330-Sound-Repository/main/A330CEON1.wav",
        "toga": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A330-Sound-Repository/main/A330CEOTOGA.wav"
      },
      "layerLpf": {
        "idle": 3500,
        "n1": 3500,
        "toga": 3000
      },
      "engineLpf": {
        "default": {
          "idle": 4000,
          "n1": 4000,
          "toga": 1300,
          "buzzsaw": 2800
        },
        "interior": {
          "cockpit": {
            "idle": 750,
            "n1": 750,
            "toga": 400,
            "buzzsaw": 750
          },
          "wing2": {
            "idle": 1200,
            "n1": 1200,
            "toga": 275,
            "buzzsaw": 650
          },
          "wingEngine": {
            "idle": 1050,
            "n1": 1050,
            "toga": 350,
            "buzzsaw": 700
          }
        }
      },
      "profiles": {
        "wingEngine": {
          "gain": 0.8925,
          "lowShelfFreq": 275,
          "lowShelfGain": 6,
          "peakFreq": 1000,
          "peakGain": -4,
          "peakQ": 1.4,
          "highShelfFreq": 2000,
          "highShelfGain": -9,
          "lpfFreq": 700
        },
        "wing2": {
          "gain": 0.7875,
          "lowShelfFreq": 300,
          "lowShelfGain": 6,
          "peakFreq": 800,
          "peakGain": -5,
          "peakQ": 0.8,
          "highShelfFreq": 1800,
          "highShelfGain": -10,
          "lpfFreq": 500
        },
        "cockpit": {
          "gain": 1,
          "lowShelfFreq": 200,
          "lowShelfGain": 0,
          "peakFreq": 1000,
          "peakGain": 0,
          "peakQ": 1,
          "highShelfFreq": 3000,
          "highShelfGain": -6,
          "lpfFreq": 1100
        },
        "exterior": {
          "gain": 1,
          "lowShelfFreq": 200,
          "lowShelfGain": 0,
          "peakFreq": 1000,
          "peakGain": 0,
          "peakQ": 1,
          "highShelfFreq": 3000,
          "highShelfGain": 0,
          "lpfFreq": 20000
        },
        "exteriorFront": {
          "gain": 1,
          "lowShelfFreq": 250,
          "lowShelfGain": -3,
          "peakFreq": 1400,
          "peakGain": 2,
          "peakQ": 1.1,
          "highShelfFreq": 2200,
          "highShelfGain": 3,
          "lpfFreq": 16000
        },
        "exteriorBehind": {
          "gain": 1,
          "lowShelfFreq": 250,
          "lowShelfGain": 4,
          "peakFreq": 900,
          "peakGain": -1.5,
          "peakQ": 0.9,
          "highShelfFreq": 2200,
          "highShelfGain": -6,
          "lpfFreq": 7000
        }
      },
      "views": {
        "cockpit": ["cockpit", "jumpseat", "jump seat", "virtual cockpit", "vc"],
        "wing2": ["wing 2", "wing2", "left wing", "right wing", "wingr2", "wingl2"],
        "side": ["left wing", "right wing", "left", "right", "wingr1", "wingl1", "business class"],
        "engine": ["eng", "trent", "700", "wingr1", "wingl1", "business class"],
        "hasWingEngine": true
      },
      "views2": {
        "cockpit": ["cockpit", "jumpseat", "jump seat", "virtual cockpit", "vc", "2d cockpit"],
        "wing2": ["wing 2", "wing2", "left wing", "right wing", "wing view", "external wing", "wingr2", "wingl2"],
        "side": ["wingr1", "wingl1", "business class"],
        "engine": ["wingr1", "wingl1", "business class"],
        "hasWingEngine": true
      },
      "mix": {
        "masterBoost": 1.8,
        "cockpitAtten": 0.42,
        "wing2Atten": 0.6,
        "preGainDb": 0,
        "compThresholdDb": -18,
        "compRatio": 2,
        "cockpitInteriorBlend": 0.75,
        "wing2InteriorBlend": 0.7,
        "exteriorInteriorBlend": 0,
        "wing2EngineAtten": 0.4725,
        "wingEngineAtten": 0.6325,
        "interiorSmoothTc": 0.025,
        "wing2VibeDepth": 0.015,
        "wing2VibeRate": 2.5,
        "exteriorAmbLevel": 0.2,
        "exteriorLpfFreq": 1800,
        "tireScreechVol": 0.45,
        "togaGain": 0.935,
        "idleGain": 1.44,
        "n1Gain": 1.265,
        "engineLayerAtten": 1.05,
        "idleWing2Boost": 1,
        "ambBoostWing": 1.455
      },
      "engine": {
        "rpmMin": 1000,
        "rpmShutdown": 950,
        "startupSuppressMs": 2000,
        "spawnProtectionMs": 8000,
        "altitudeProtectionFt": 1000,
        "idleUnfiltered": false
      },
      "curve": {
        "idleCut": 0.45,
        "n1Start": 0.12,
        "n1Full": 0.52,
        "togaStart": 0.42,
        "midCenter": 0.33,
        "midWidth": 0.32,
        "midGain": 2,
        "floor": 0.1,
        "togaExp": 1.15,
        "bellExp": 1.6,
        "idleBell": 0.65,
        "n1Bell": "bell",
        "togaPow": 1.05,
        "idleFloorMul": 1.25,
        "idleFinalMul": 1,
        "n1FloorMul": 0.95,
        "togaBoost": 1,
        "togaFinalMul": null,
        "idleRateMul": 9,
        "idleRateTail": 0.9,
        "n1RateK": 0.22,
        "togaRateK": 0.5,
        "togaRateExp": 1.05,
        "idlePower": 0.7,
        "idlePitchIntensity": 0.4,
        "idleVolumeBoost": 0.5,
        "n1Scale": 0.16,
        "maxLayerGain": 1.8,
        "n1BellMode": "bell",
        "n1BellMul": 1
      },
      "buzzsaw": {
        "lpf": 2800,
        "altMaxFt": 8000,
        "altExp": 1.6,
        "altScale": 1.5,
        "throttleStart": 0.65,
        "throttleEnd": 1,
        "rateBase": 0.78,
        "rateSpan": 0.2,
        "wingEngineBoost": 1,
        "wing2Atten": 1,
        "cockpitAtten": 1
      },
      "toga": {
        "altVolume": null,
        "cockpitMult": 1,
        "altitudeHF": true
      },
      "gear": {
        "lpfFreq": 1400,
        "lpfQ": 0.6,
        "fadeIn": 0.4,
        "fadeOut": 0.4,
        "volCockpit": 0.633,
        "volWing2": 0.437,
        "volWingEngine": 0.748
      },
      "assets": {
        "ambWing2Air": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A320-CEO-Family-Sound-Pack-Official/main/freesound_community-airplane-interior-ambience-59644.mp3",
        "ambWing2": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/freesound_community-plane-interior-52034%20(1).mp3",
        "ambExterior": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Weather_%23.wav",
        "gearClick": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Master_9.wav",
        "spoiler": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-Sound-Pack-Official/main/Master_10.wav",
        "engineToggle": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Master_19.wav",
        "rain": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Airbus-A320-Sound-Pack/main/Weather_6.wav",
        "tire": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-Sound-Pack-Official/main/freesound_community-cinematic-deep-rumble-6418.mp3",
        "tireScreech": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Sound-Pack-Ultimate-/main/Tire%20Screech.wav",
        "rattle": {
          "low": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Dynamics_20.wav",
          "midL": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Dynamics_2.wav",
          "transC": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Dynamics_18.wav"
        },
        "startupExt": "https://raw.githubusercontent.com/ChristianPilotAlex003/A330-Sound-Repository/main/A330STARTUPSOUND.wav",
        "startupInt": "https://raw.githubusercontent.com/ChristianPilotAlex003/A330-Sound-Repository/main/A330INTERIORCABINSTARTUP.wav",
        "shutdown": "https://raw.githubusercontent.com/ChristianPilotAlex003/A330-Sound-Repository/main/A330ShutDownSound.wav",
        "ambCockpit": "https://raw.githubusercontent.com/ChristianPilotAlex003/A350-Sound-Pack-For-GeoFS/main/A350CockpitBackgroundNoise.mp3",
        "buzzsaw": "",
        "gearHum": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-Sound-Pack-Official/main/Gear%20Down%20(3).wav",
        "tray": "",
        "flapClick": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A320NEO-Sound-Repository/main/AirbusFlapSound.wav",
        "flapMotor": null,
        "flapHum": "https://raw.githubusercontent.com/ChristianPilotAlex003/A330-Sound-Repository/main/A330CEOFLAPSOUND.wav",
        "configAlarm": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Sound-Pack-Ultimate-/main/Airbus%20Alarm.wav"
      }
    },
    // ---- A330-900neo ---------------------------------------------------
    {
      "code": "a339",
      "suffix": "A339",
      "name": "A330-900neo",
      "ids": ["4631"],
      "layers": {
        "idle": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A330NEO-Sound-Repository/main/A339IDLE.wav",
        "n1": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A330NEO-Sound-Repository/main/A339N1.wav",
        "toga": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A330NEO-Sound-Repository/main/A339TOGA.wav"
      },
      "layerLpf": {
        "idle": 4000,
        "n1": 4000,
        "toga": 1300
      },
      "engineLpf": {
        "default": {
          "idle": 4000,
          "n1": 4000,
          "toga": 1300,
          "buzzsaw": 2800
        },
        "interior": {
          "cockpit": {
            "idle": 1500,
            "n1": 1500,
            "toga": 800,
            "buzzsaw": 1500
          },
          "wing2": {
            "idle": 3700,
            "n1": 3700,
            "toga": 1000,
            "buzzsaw": 2200
          },
          "wingEngine": {
            "idle": 2800,
            "n1": 2800,
            "toga": 950,
            "buzzsaw": 2000
          }
        }
      },
      "profiles": {
        "wingEngine": {
          "gain": 0.65,
          "lowShelfFreq": 275,
          "lowShelfGain": 6,
          "peakFreq": 1000,
          "peakGain": -4,
          "peakQ": 1.4,
          "highShelfFreq": 2000,
          "highShelfGain": -9,
          "lpfFreq": 2200
        },
        "wing2": {
          "gain": 0.55,
          "lowShelfFreq": 300,
          "lowShelfGain": 6,
          "peakFreq": 800,
          "peakGain": -5,
          "peakQ": 0.8,
          "highShelfFreq": 1800,
          "highShelfGain": -10,
          "lpfFreq": 2200
        },
        "cockpit": {
          "gain": 1,
          "lowShelfFreq": 200,
          "lowShelfGain": 0,
          "peakFreq": 1000,
          "peakGain": 0,
          "peakQ": 1,
          "highShelfFreq": 3000,
          "highShelfGain": -6,
          "lpfFreq": 2200
        },
        "exterior": {
          "gain": 1,
          "lowShelfFreq": 200,
          "lowShelfGain": 0,
          "peakFreq": 1000,
          "peakGain": 0,
          "peakQ": 1,
          "highShelfFreq": 3000,
          "highShelfGain": 0,
          "lpfFreq": 20000
        },
        "exteriorFront": {
          "gain": 1,
          "lowShelfFreq": 250,
          "lowShelfGain": -3,
          "peakFreq": 1400,
          "peakGain": 2,
          "peakQ": 1.1,
          "highShelfFreq": 2200,
          "highShelfGain": 3,
          "lpfFreq": 16000
        },
        "exteriorBehind": {
          "gain": 1,
          "lowShelfFreq": 250,
          "lowShelfGain": 4,
          "peakFreq": 900,
          "peakGain": -1.5,
          "peakQ": 0.9,
          "highShelfFreq": 2200,
          "highShelfGain": -6,
          "lpfFreq": 7000
        }
      },
      "views": {
        "cockpit": ["cockpit", "jumpseat", "jump seat", "virtual cockpit", "vc"],
        "wing2": ["wing 2", "wing2", "left wing", "right wing"],
        "side": ["left wing", "right wing", "left", "right"],
        "engine": ["eng", "trent", "700"],
        "hasWingEngine": true
      },
      "views2": {
        "cockpit": ["cockpit", "jumpseat", "jump seat", "virtual cockpit", "vc", "2d cockpit"],
        "wing2": ["wing 2", "wing2", "left wing", "right wing", "wing view", "external wing"],
        "side": ["left wing", "right wing", "left", "right"],
        "engine": ["eng", "trent", "700"],
        "hasWingEngine": true
      },
      "mix": {
        "masterBoost": 1.8,
        "cockpitAtten": 0.42,
        "wing2Atten": 0.6,
        "preGainDb": 0,
        "compThresholdDb": -18,
        "compRatio": 2,
        "cockpitInteriorBlend": 0.75,
        "wing2InteriorBlend": 0.7,
        "exteriorInteriorBlend": 0,
        "wing2EngineAtten": 0.45,
        "wingEngineAtten": 0.55,
        "interiorSmoothTc": 0.025,
        "wing2VibeDepth": 0.015,
        "wing2VibeRate": 2.5,
        "exteriorAmbLevel": 0.2,
        "exteriorLpfFreq": 1800,
        "tireScreechVol": 0.45,
        "togaGain": 1.15,
        "engineLayerAtten": 1,
        "idleWing2Boost": 1
      },
      "engine": {
        "rpmMin": 1000,
        "rpmShutdown": 950,
        "startupSuppressMs": 2000,
        "spawnProtectionMs": 8000,
        "altitudeProtectionFt": 1000,
        "idleUnfiltered": false
      },
      "curve": {
        "idleCut": 0.45,
        "n1Start": 0.12,
        "n1Full": 0.52,
        "togaStart": 0.42,
        "midCenter": 0.33,
        "midWidth": 0.32,
        "midGain": 2,
        "floor": 0.1,
        "togaExp": 1.15,
        "bellExp": 1.6,
        "idleBell": 0.65,
        "n1Bell": "bell",
        "togaPow": 1.05,
        "idleFloorMul": 1.25,
        "idleFinalMul": 1,
        "n1FloorMul": 0.95,
        "togaBoost": 1,
        "togaFinalMul": null,
        "idleRateMul": 9,
        "idleRateTail": 0.9,
        "n1RateK": 0.22,
        "togaRateK": 0.5,
        "togaRateExp": 1.05,
        "idlePower": 0.7,
        "idlePitchIntensity": 0.4,
        "idleVolumeBoost": 0.5,
        "n1Scale": 0.5,
        "maxLayerGain": 1.8,
        "n1BellMode": "bell",
        "n1BellMul": 1
      },
      "buzzsaw": {
        "lpf": 2800,
        "altMaxFt": 8000,
        "altExp": 1.6,
        "altScale": 1.5,
        "throttleStart": 0.65,
        "throttleEnd": 1,
        "rateBase": 0.78,
        "rateSpan": 0.2,
        "wingEngineBoost": 1,
        "wing2Atten": 1,
        "cockpitAtten": 1
      },
      "toga": {
        "altVolume": null,
        "cockpitMult": 1,
        "altitudeHF": true
      },
      "gear": {
        "lpfFreq": 1400,
        "lpfQ": 0.6,
        "fadeIn": 0.4,
        "fadeOut": 0.4,
        "volCockpit": 0.633,
        "volWing2": 0.437,
        "volWingEngine": 0.748
      },
      "assets": {
        "ambWing2Air": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A320-CEO-Family-Sound-Pack-Official/main/freesound_community-airplane-interior-ambience-59644.mp3",
        "ambWing2": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/freesound_community-plane-interior-52034%20(1).mp3",
        "ambExterior": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Weather_%23.wav",
        "gearClick": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Master_9.wav",
        "spoiler": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-Sound-Pack-Official/main/Master_10.wav",
        "engineToggle": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Master_19.wav",
        "rain": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Airbus-A320-Sound-Pack/main/Weather_6.wav",
        "tire": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-Sound-Pack-Official/main/freesound_community-cinematic-deep-rumble-6418.mp3",
        "tireScreech": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Sound-Pack-Ultimate-/main/Tire%20Screech.wav",
        "rattle": {
          "low": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Dynamics_20.wav",
          "midL": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Dynamics_2.wav",
          "transC": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Dynamics_18.wav"
        },
        "startupExt": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A330NEO-Sound-Repository/main/A339STARTUP.wav",
        "startupInt": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A330NEO-Sound-Repository/main/A339STARTUPINT.wav",
        "shutdown": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A330NEO-Sound-Repository/main/A339SHUTDOWN.wav",
        "ambCockpit": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A330NEO-Sound-Repository/main/A339COCKPITAMBIENCE.wav",
        "buzzsaw": "",
        "gearHum": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-Sound-Pack-Official/main/Gear%20Down%20(3).wav",
        "tray": "",
        "flapClick": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A320NEO-Sound-Repository/main/AirbusFlapSound.wav",
        "flapMotor": null,
        "flapHum": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A330NEO-Sound-Repository/main/A339FLAP.wav",
        "configAlarm": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Sound-Pack-Ultimate-/main/Airbus%20Alarm.wav"
      }
    },
    // ---- A380 ----------------------------------------------------------
    {
      "code": "a380",
      "suffix": "A380",
      "name": "A380",
      "ids": ["10"],
      "layers": {
        "idle": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A380-Sound-Repository/main/A380%20IDLE.wav",
        "n1": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A380-Sound-Repository/main/A380N1.wav",
        "toga": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A380-Sound-Repository/main/A380%20TOGA%201.wav"
      },
      "layerLpf": {
        "idle": 20000,
        "n1": 20000,
        "toga": 20000
      },
      "engineLpf": {
        "default": {
          "idle": 4000,
          "n1": 4000,
          "toga": 1300,
          "buzzsaw": 2800
        },
        "interior": {
          "cockpit": {
            "idle": 1500,
            "n1": 1500,
            "toga": 800,
            "buzzsaw": 1500
          },
          "wing2": {
            "idle": 2800,
            "n1": 2800,
            "toga": 600,
            "buzzsaw": 1500
          },
          "wingEngine": {
            "idle": 4000,
            "n1": 4000,
            "toga": 1300,
            "buzzsaw": 2800
          }
        }
      },
      "profiles": {
        "wingEngine": {
          "gain": 0.65,
          "lowShelfFreq": 275,
          "lowShelfGain": 6,
          "peakFreq": 1000,
          "peakGain": -4,
          "peakQ": 1.4,
          "highShelfFreq": 2000,
          "highShelfGain": -9,
          "lpfFreq": 2200
        },
        "wing2": {
          "gain": 0.9,
          "lowShelfFreq": 300,
          "lowShelfGain": 6,
          "peakFreq": 800,
          "peakGain": -5,
          "peakQ": 0.8,
          "highShelfFreq": 2000,
          "highShelfGain": -10,
          "lpfFreq": 2400
        },
        "cockpit": {
          "gain": 1,
          "lowShelfFreq": 200,
          "lowShelfGain": 0,
          "peakFreq": 1000,
          "peakGain": 0,
          "peakQ": 1,
          "highShelfFreq": 3000,
          "highShelfGain": -6,
          "lpfFreq": 2200
        },
        "exterior": {
          "gain": 1,
          "lowShelfFreq": 200,
          "lowShelfGain": 0,
          "peakFreq": 1000,
          "peakGain": 0,
          "peakQ": 1,
          "highShelfFreq": 3000,
          "highShelfGain": 0,
          "lpfFreq": 20000
        },
        "exteriorFront": {
          "gain": 1,
          "lowShelfFreq": 250,
          "lowShelfGain": -3,
          "peakFreq": 1400,
          "peakGain": 2,
          "peakQ": 1.1,
          "highShelfFreq": 2200,
          "highShelfGain": 3,
          "lpfFreq": 16000
        },
        "exteriorBehind": {
          "gain": 1,
          "lowShelfFreq": 250,
          "lowShelfGain": 4,
          "peakFreq": 900,
          "peakGain": -1.5,
          "peakQ": 0.9,
          "highShelfFreq": 2200,
          "highShelfGain": -6,
          "lpfFreq": 7000
        }
      },
      "views": {
        "cockpit": ["cockpit", "jumpseat", "jump seat", "virtual cockpit", "vc"],
        "wing2": ["wing 2", "wing2", "left wing", "right wing"],
        "side": ["left wing", "right wing", "left", "right"],
        "engine": ["eng", "trent", "700"],
        "hasWingEngine": true
      },
      "views2": {
        "cockpit": ["cockpit", "jumpseat", "jump seat", "virtual cockpit", "vc", "2d cockpit"],
        "wing2": ["wing 2", "wing2", "left wing", "right wing", "wing view", "external wing"],
        "side": [],
        "engine": [],
        "hasWingEngine": false
      },
      "mix": {
        "masterBoost": 2.2,
        "cockpitAtten": 0.6,
        "wing2Atten": 0.9,
        "preGainDb": 0,
        "compThresholdDb": -18,
        "compRatio": 2,
        "cockpitInteriorBlend": 0.75,
        "wing2InteriorBlend": 0.7,
        "exteriorInteriorBlend": 0,
        "wing2EngineAtten": 0.85,
        "wingEngineAtten": 0.55,
        "interiorSmoothTc": 0.025,
        "wing2VibeDepth": 0.015,
        "wing2VibeRate": 2.5,
        "exteriorAmbLevel": 0.2,
        "exteriorLpfFreq": 1800,
        "tireScreechVol": 0.45,
        "togaGain": 1.15,
        "engineLayerAtten": 1,
        "idleWing2Boost": 1
      },
      "engine": {
        "rpmMin": 1000,
        "rpmShutdown": 950,
        "startupSuppressMs": 2000,
        "spawnProtectionMs": 8000,
        "altitudeProtectionFt": 1000,
        "idleUnfiltered": false
      },
      "curve": {
        "idleCut": 0.45,
        "n1Start": 0.2,
        "n1Full": 0.52,
        "togaStart": 0.6,
        "midCenter": 0.33,
        "midWidth": 0.32,
        "midGain": 2,
        "floor": 0.1,
        "togaExp": 1.15,
        "bellExp": 1.6,
        "idleBell": 0.05,
        "n1Bell": 0.35,
        "togaPow": 1.15,
        "idleFloorMul": 1.15,
        "idleFinalMul": 0.6,
        "n1FloorMul": 0.55,
        "togaBoost": 0.75,
        "togaFinalMul": 0.75,
        "idleRateMul": 9,
        "idleRateTail": 0.9,
        "n1RateK": 0.24,
        "togaRateK": 0.48,
        "togaRateExp": 1.05,
        "idlePower": 0.65,
        "idlePitchIntensity": 0.5,
        "idleVolumeBoost": 0.45,
        "n1Scale": 0.38,
        "maxLayerGain": 1.8,
        "n1BellMode": "const",
        "n1BellMul": 0.35
      },
      "buzzsaw": {
        "lpf": 3200,
        "altMaxFt": 8000,
        "altExp": 1.6,
        "altScale": 1.5,
        "throttleStart": 0.65,
        "throttleEnd": 1,
        "rateBase": 0.78,
        "rateSpan": 0.2,
        "wingEngineBoost": 1,
        "wing2Atten": 1,
        "cockpitAtten": 1
      },
      "toga": {
        "altVolume": null,
        "cockpitMult": 1,
        "altitudeHF": true
      },
      "gear": {
        "lpfFreq": 1400,
        "lpfQ": 0.6,
        "fadeIn": 0.4,
        "fadeOut": 0.4,
        "volCockpit": 0.34,
        "volWing2": 0.298,
        "volWingEngine": 0.636
      },
      "assets": {
        "ambWing2Air": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A320-CEO-Family-Sound-Pack-Official/main/freesound_community-airplane-interior-ambience-59644.mp3",
        "ambWing2": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/freesound_community-plane-interior-52034%20(1).mp3",
        "ambExterior": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Weather_%23.wav",
        "gearClick": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Master_9.wav",
        "spoiler": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-Sound-Pack-Official/main/Master_10.wav",
        "engineToggle": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Master_19.wav",
        "rain": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Airbus-A320-Sound-Pack/main/Weather_6.wav",
        "tire": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-Sound-Pack-Official/main/freesound_community-cinematic-deep-rumble-6418.mp3",
        "tireScreech": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Sound-Pack-Ultimate-/main/Tire%20Screech.wav",
        "rattle": {
          "low": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Dynamics_20.wav",
          "midL": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Dynamics_2.wav",
          "transC": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Dynamics_18.wav"
        },
        "startupExt": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A380-Sound-Repository/main/A380STARTUP.wav",
        "startupInt": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A380-Sound-Repository/main/A380STARTUPINTERIOR.wav",
        "shutdown": "https://raw.githubusercontent.com/ChristianPilotAlex003/A330-Sound-Repository/main/A330ShutDownSound.wav",
        "ambCockpit": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A380-Sound-Repository/main/A380%20Ambience.wav",
        "buzzsaw": "",
        "gearHum": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-Sound-Pack-Official/main/Gear%20Down%20(3).wav",
        "tray": "",
        "flapClick": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A320NEO-Sound-Repository/main/AirbusFlapSound.wav",
        "flapMotor": null,
        "flapHum": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A380-Sound-Repository/main/A380FLAP.wav",
        "configAlarm": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Sound-Pack-Ultimate-/main/Airbus%20Alarm.wav"
      }
    },
    // ---- A340 ----------------------------------------------------------
    {
      "code": "a340",
      "suffix": "A340",
      "name": "A340",
      "ids": ["6006", "2153", "5998"],
      "layers": {
        "idle": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A340-Sound-Repository/main/A340IDLE.wav",
        "n1": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A340-Sound-Repository/main/A340N1.wav",
        "toga": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A340-Sound-Repository/main/A340TOGA.wav"
      },
      "layerLpf": {
        "idle": 20000
      },
      "engineLpf": {
        "default": {
          "idle": 20000,
          "n1": 6000,
          "toga": 10000,
          "buzzsaw": 10000
        },
        "interior": {
          "cockpit": {
            "idle": 10000,
            "n1": 750,
            "toga": 400,
            "buzzsaw": 750
          },
          "wing2": {
            "idle": 10000,
            "n1": 2100,
            "toga": 350,
            "buzzsaw": 850
          },
          "wingEngine": {
            "idle": 10000,
            "n1": 3000,
            "toga": 5000,
            "buzzsaw": 5000
          }
        }
      },
      "profiles": {
        "wingEngine": {
          "gain": 0.95,
          "lowShelfFreq": 275,
          "lowShelfGain": 6,
          "peakFreq": 900,
          "peakGain": -4,
          "peakQ": 1.1,
          "highShelfFreq": 1900,
          "highShelfGain": -6,
          "lpfFreq": 1650
        },
        "wing2": {
          "gain": 0.55,
          "lowShelfFreq": 300,
          "lowShelfGain": 6,
          "peakFreq": 800,
          "peakGain": -5,
          "peakQ": 0.8,
          "highShelfFreq": 1800,
          "highShelfGain": -8,
          "lpfFreq": 1500
        },
        "cockpit": {
          "gain": 1,
          "lowShelfFreq": 200,
          "lowShelfGain": 0,
          "peakFreq": 1000,
          "peakGain": 0,
          "peakQ": 1,
          "highShelfFreq": 3000,
          "highShelfGain": 0,
          "lpfFreq": 10000
        },
        "exterior": {
          "gain": 1,
          "lowShelfFreq": 200,
          "lowShelfGain": 0,
          "peakFreq": 1000,
          "peakGain": 0,
          "peakQ": 1,
          "highShelfFreq": 3000,
          "highShelfGain": 0,
          "lpfFreq": 20000
        },
        "exteriorFront": {
          "gain": 1,
          "lowShelfFreq": 250,
          "lowShelfGain": -3,
          "peakFreq": 1400,
          "peakGain": 2,
          "peakQ": 1.1,
          "highShelfFreq": 2200,
          "highShelfGain": 3,
          "lpfFreq": 16000
        },
        "exteriorBehind": {
          "gain": 1,
          "lowShelfFreq": 250,
          "lowShelfGain": 4,
          "peakFreq": 900,
          "peakGain": -1.5,
          "peakQ": 0.9,
          "highShelfFreq": 2200,
          "highShelfGain": -6,
          "lpfFreq": 7000
        }
      },
      "views": {
        "cockpit": ["cockpit", "copilot", "jumpseat", "jump seat", "virtual cockpit", "vc"],
        "wing2": ["wing 2", "wing2", "left wing", "right wing"],
        "side": ["left wing", "right wing", "left", "right"],
        "engine": ["cfm", "iaev", "v2500", "engines"],
        "hasWingEngine": true
      },
      "views2": {
        "cockpit": ["cockpit", "jumpseat", "jump seat", "virtual cockpit", "vc", "2d cockpit"],
        "wing2": ["wing 2", "wing2", "left wing", "right wing", "wing view", "external wing"],
        "side": [],
        "engine": [],
        "hasWingEngine": false
      },
      "mix": {
        "masterBoost": 1.8,
        "cockpitAtten": 0.35,
        "wing2Atten": 0.65,
        "preGainDb": -2,
        "compThresholdDb": -8,
        "compRatio": 1,
        "cockpitInteriorBlend": 0.95,
        "wing2InteriorBlend": 0.9,
        "exteriorInteriorBlend": 0,
        "wing2EngineAtten": 0.55,
        "wingEngineAtten": 1,
        "interiorSmoothTc": 0.025,
        "wing2VibeDepth": 0.015,
        "wing2VibeRate": 2.5,
        "exteriorAmbLevel": 0.2,
        "exteriorLpfFreq": 1800,
        "tireScreechVol": 0.45,
        "togaGain": 1.15,
        "engineLayerAtten": 1.2,
        "idleWing2Boost": 1
      },
      "engine": {
        "rpmMin": 1000,
        "rpmShutdown": 990,
        "startupSuppressMs": 2000,
        "spawnProtectionMs": 8000,
        "altitudeProtectionFt": 1000,
        "idleUnfiltered": true
      },
      "curve": {
        "idleCut": 0.45,
        "n1Start": 0.12,
        "n1Full": 0.52,
        "togaStart": 0.5,
        "midCenter": 0.33,
        "midWidth": 0.32,
        "midGain": 2,
        "floor": 0.1,
        "togaExp": 1.25,
        "bellExp": 1.6,
        "idleBell": 0.85,
        "n1Bell": 5,
        "togaPow": 1,
        "idleFloorMul": 1,
        "idleFinalMul": 0.75,
        "n1FloorMul": 2,
        "togaBoost": 0.4,
        "togaFinalMul": null,
        "idleRateMul": 9,
        "idleRateTail": 0.9,
        "n1RateK": 0.22,
        "togaRateK": 0.4,
        "togaRateExp": 1.15,
        "idlePower": 0.8,
        "idlePitchIntensity": 0.26,
        "idleVolumeBoost": 1,
        "n1Scale": 0.6,
        "maxLayerGain": 0.8,
        "n1BellMode": "const",
        "n1BellMul": 5
      },
      "buzzsaw": {
        "lpf": 10000,
        "altMaxFt": 8000,
        "altExp": 1.6,
        "altScale": 1.5,
        "throttleStart": 0.65,
        "throttleEnd": 1,
        "rateBase": 0.78,
        "rateSpan": 0.2,
        "wingEngineBoost": 1,
        "wing2Atten": 1,
        "cockpitAtten": 1
      },
      "toga": {
        "altVolume": null,
        "cockpitMult": 1,
        "altitudeHF": false
      },
      "gear": {
        "lpfFreq": 1400,
        "lpfQ": 0.6,
        "fadeIn": 0.4,
        "fadeOut": 0.4,
        "volCockpit": 0.633,
        "volWing2": 0.437,
        "volWingEngine": 0.748
      },
      "assets": {
        "ambWing2Air": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A320-CEO-Family-Sound-Pack-Official/main/freesound_community-airplane-interior-ambience-59644.mp3",
        "ambWing2": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/freesound_community-plane-interior-52034%20(1).mp3",
        "ambExterior": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Weather_%23.wav",
        "gearClick": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Master_9.wav",
        "spoiler": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-Sound-Pack-Official/main/Master_10.wav",
        "engineToggle": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Master_19.wav",
        "rain": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Airbus-A320-Sound-Pack/main/Weather_6.wav",
        "tire": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-Sound-Pack-Official/main/freesound_community-cinematic-deep-rumble-6418.mp3",
        "tireScreech": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Sound-Pack-Ultimate-/main/Tire%20Screech.wav",
        "rattle": {
          "low": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Dynamics_20.wav",
          "midL": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Dynamics_2.wav",
          "transC": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Dynamics_18.wav"
        },
        "startupExt": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A340-Sound-Repository/main/A340STARTUP.wav",
        "startupInt": "https://raw.githubusercontent.com/ChristianPilotAlex003/A350-Sound-Pack-For-GeoFS/main/Engine_27.wav",
        "shutdown": "https://raw.githubusercontent.com/ChristianPilotAlex003/A350-Sound-Pack-For-GeoFS/main/A350ShutdownSound.mp3",
        "ambCockpit": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B737NG-Family-Sound-Pack/main/B737CockpitAmbience.wav",
        "buzzsaw": "",
        "gearHum": "",
        "tray": "",
        "flapClick": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A320NEO-Sound-Repository/main/AirbusFlapSound.wav",
        "flapMotor": null,
        "flapHum": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A340-Sound-Repository/main/A340FlapSound.wav",
        "configAlarm": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Sound-Pack-Ultimate-/main/Airbus%20Alarm.wav"
      }
    },
    // ---- A350 ----------------------------------------------------------
    {
      "code": "a350",
      "suffix": "A350",
      "name": "A350",
      "ids": ["24", "2973", "239"],
      "layers": {
        "idle": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A350-Sound-Repository/main/A350IDLE.wav",
        "n1": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A350-Sound-Repository/main/A350N1.wav",
        "toga": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A350-Sound-Repository/main/A350TOGA.wav"
      },
      "layerLpf": {
        "idle": 10000,
        "n1": 10000,
        "toga": 10000
      },
      "engineLpf": {
        "default": {
          "idle": 4000,
          "n1": 5300,
          "toga": 5300,
          "buzzsaw": 2800
        },
        "interior": {
          "cockpit": {
            "idle": 1350,
            "n1": 1350,
            "toga": 1350,
            "buzzsaw": 1350
          },
          "wing2": {
            "idle": 1100,
            "n1": 1400,
            "toga": 400,
            "buzzsaw": 700
          },
          "wingEngine": {
            "idle": 4000,
            "n1": 5300,
            "toga": 5300,
            "buzzsaw": 2800
          }
        }
      },
      "profiles": {
        "wingEngine": {
          "gain": 1,
          "lowShelfFreq": 200,
          "lowShelfGain": 0,
          "peakFreq": 1000,
          "peakGain": 0,
          "peakQ": 1,
          "highShelfFreq": 3000,
          "highShelfGain": 0,
          "lpfFreq": 20000
        },
        "wing2": {
          "gain": 1,
          "lowShelfFreq": 200,
          "lowShelfGain": 0,
          "peakFreq": 1000,
          "peakGain": 0,
          "peakQ": 1,
          "highShelfFreq": 3000,
          "highShelfGain": 0,
          "lpfFreq": 20000
        },
        "cockpit": {
          "gain": 1,
          "lowShelfFreq": 200,
          "lowShelfGain": 0,
          "peakFreq": 1000,
          "peakGain": 0,
          "peakQ": 1,
          "highShelfFreq": 3000,
          "highShelfGain": 0,
          "lpfFreq": 20000
        },
        "exterior": {
          "gain": 1,
          "lowShelfFreq": 200,
          "lowShelfGain": 0,
          "peakFreq": 1000,
          "peakGain": 0,
          "peakQ": 1,
          "highShelfFreq": 3000,
          "highShelfGain": 0,
          "lpfFreq": 20000
        },
        "exteriorFront": {
          "gain": 1,
          "lowShelfFreq": 250,
          "lowShelfGain": -3,
          "peakFreq": 1400,
          "peakGain": 2,
          "peakQ": 1.1,
          "highShelfFreq": 2200,
          "highShelfGain": 3,
          "lpfFreq": 16000
        },
        "exteriorBehind": {
          "gain": 1,
          "lowShelfFreq": 250,
          "lowShelfGain": 4,
          "peakFreq": 900,
          "peakGain": -1.5,
          "peakQ": 0.9,
          "highShelfFreq": 2200,
          "highShelfGain": -6,
          "lpfFreq": 7000
        }
      },
      "views": {
        "cockpit": ["cockpit", "jumpseat", "jump seat", "virtual cockpit", "vc"],
        "wing2": ["wing 2", "wing2", "left wing", "right wing"],
        "side": [],
        "engine": [],
        "hasWingEngine": false
      },
      "views2": {
        "cockpit": ["cockpit", "jumpseat", "jump seat", "virtual cockpit", "vc", "2d cockpit"],
        "wing2": ["wing 2", "wing2", "left wing", "right wing", "wing view", "external wing"],
        "side": [],
        "engine": [],
        "hasWingEngine": false
      },
      "mix": {
        "masterBoost": 2,
        "cockpitAtten": 0.65,
        "wing2Atten": 1.08,
        "preGainDb": 0,
        "compThresholdDb": -18,
        "compRatio": 2,
        "cockpitInteriorBlend": 0.75,
        "wing2InteriorBlend": 0.7,
        "exteriorInteriorBlend": 0,
        "wing2EngineAtten": 0.55,
        "wingEngineAtten": 0.55,
        "interiorSmoothTc": 0.025,
        "wing2VibeDepth": 0.015,
        "wing2VibeRate": 2.5,
        "exteriorAmbLevel": 0.2,
        "exteriorLpfFreq": 1800,
        "tireScreechVol": 0.45,
        "togaGain": 1.15,
        "engineLayerAtten": 1,
        "idleWing2Boost": 1
      },
      "engine": {
        "rpmMin": 1000,
        "rpmShutdown": 900,
        "startupSuppressMs": 2000,
        "spawnProtectionMs": 8000,
        "altitudeProtectionFt": 1000,
        "idleUnfiltered": false
      },
      "curve": {
        "idleCut": 0.45,
        "n1Start": 0.12,
        "n1Full": 0.52,
        "togaStart": 0.42,
        "midCenter": 0.33,
        "midWidth": 0.32,
        "midGain": 2,
        "floor": 0.1,
        "togaExp": 1.15,
        "bellExp": 1.6,
        "idleBell": 0.55,
        "togaPow": 1.05,
        "idleFloorMul": 1.25,
        "idleFinalMul": 1,
        "n1FloorMul": 1.5,
        "togaBoost": 1,
        "togaFinalMul": null,
        "idleRateMul": 9,
        "idleRateTail": 0.9,
        "n1RateK": 0.22,
        "togaRateK": 0.5,
        "togaRateExp": 1.05,
        "idlePower": 0.5,
        "idlePitchIntensity": 0.7,
        "idleVolumeBoost": 0.5,
        "n1Scale": 0.35,
        "maxLayerGain": 1.8,
        "n1BellMode": "bell",
        "n1BellMul": 1.5
      },
      "buzzsaw": {
        "lpf": 2800,
        "altMaxFt": 8000,
        "altExp": 1.6,
        "altScale": 0.38249999999999995,
        "throttleStart": 0.65,
        "throttleEnd": 1,
        "rateBase": 0.78,
        "rateSpan": 0.2,
        "wingEngineBoost": 1,
        "wing2Atten": 0.26,
        "cockpitAtten": 0.338
      },
      "toga": {
        "altVolume": null,
        "cockpitMult": 1,
        "altitudeHF": false
      },
      "gear": {
        "lpfFreq": 1400,
        "lpfQ": 0.6,
        "fadeIn": 0.4,
        "fadeOut": 0.4,
        "volCockpit": 0.633,
        "volWing2": 0.437,
        "volWingEngine": 0.748
      },
      "assets": {
        "ambWing2Air": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A320-CEO-Family-Sound-Pack-Official/main/freesound_community-airplane-interior-ambience-59644.mp3",
        "ambWing2": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/freesound_community-plane-interior-52034%20(1).mp3",
        "ambExterior": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Weather_%23.wav",
        "gearClick": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Master_9.wav",
        "spoiler": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-Sound-Pack-Official/main/Master_10.wav",
        "engineToggle": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Master_19.wav",
        "rain": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Airbus-A320-Sound-Pack/main/Weather_6.wav",
        "tire": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-Sound-Pack-Official/main/freesound_community-cinematic-deep-rumble-6418.mp3",
        "tireScreech": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Sound-Pack-Ultimate-/main/Tire%20Screech.wav",
        "rattle": {
          "low": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Dynamics_20.wav",
          "midL": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Dynamics_2.wav",
          "transC": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Dynamics_18.wav"
        },
        "startupExt": function(){ return String(window.geofs?.aircraft?.instance?.id || '') === '2973' ? "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-Sound-Pack-Official/main/A350-1000StartupSound.wav" : "https://raw.githubusercontent.com/ChristianPilotAlex003/A350-Sound-Pack-For-GeoFS/main/A350StartUpSound.mp3"; },
        "startupInt": "https://raw.githubusercontent.com/ChristianPilotAlex003/A350-Sound-Pack-For-GeoFS/main/Engine_27.wav",
        "shutdown": "https://raw.githubusercontent.com/ChristianPilotAlex003/A350-Sound-Pack-For-GeoFS/main/A350ShutdownSound.mp3",
        "ambCockpit": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A350-Sound-Repository/main/Dynamics_33.wav",
        "buzzsaw": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A350-Sound-Pack-Official/main/Master_17.wav",
        "gearHum": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-Sound-Pack-Official/main/Gear%20Down%20(3).wav",
        "tray": "https://raw.githubusercontent.com/ChristianPilotAlex003/A350-Sound-Pack-For-GeoFS/main/Cockpit%20Animation%20Sound.mp3",
        "flapClick": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A320NEO-Sound-Repository/main/AirbusFlapSound.wav",
        "flapMotor": {
          "start": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A350-Sound-Repository/main/A350FLAPSTART.wav",
          "loop": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A350-Sound-Repository/main/A350FLAPMID.wav",
          "end": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A350-Sound-Repository/main/A350FLAPEND%20(1).wav",
          "crossfade": 0.12
        },
        "flapHum": "https://raw.githubusercontent.com/ChristianPilotAlex003/A350-Sound-Pack-For-GeoFS/main/A350FlapSound.mp3",
        "configAlarm": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Sound-Pack-Ultimate-/main/Airbus%20Alarm.wav"
      }
    },
    // ---- B737 NG -------------------------------------------------------
    {
      "code": "b737",
      "suffix": "B737",
      "name": "B737 NG",
      "ids": ["4", "3054", "5203"],
      "layers": {
        "idle": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B737NG-Family-Sound-Pack/main/B737IDLE.wav",
        "n1": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B737NG-Family-Sound-Pack/main/B737N1.wav",
        "toga": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B737NG-Family-Sound-Pack/main/B737TOGA.wav"
      },
      "layerLpf": {
        "idle": 4000,
        "n1": 4500,
        "toga": 6000
      },
      "engineLpf": {
        "default": {
          "idle": 4000,
          "n1": 4500,
          "toga": 6000,
          "buzzsaw": 3200
        },
        "interior": {
          "cockpit": {
            "idle": 1500,
            "n1": 1500,
            "toga": 800,
            "buzzsaw": 1500
          },
          "wing2": {
            "idle": 3200,
            "n1": 3400,
            "toga": 700,
            "buzzsaw": 1700
          },
          "wingEngine": {
            "idle": 4000,
            "n1": 4500,
            "toga": 6000,
            "buzzsaw": 3200
          }
        }
      },
      "profiles": {
        "wingEngine": {
          "gain": 0.65,
          "lowShelfFreq": 275,
          "lowShelfGain": 6,
          "peakFreq": 1000,
          "peakGain": -4,
          "peakQ": 1.3,
          "highShelfFreq": 2000,
          "highShelfGain": -6,
          "lpfFreq": 3800
        },
        "wing2": {
          "gain": 1,
          "lowShelfFreq": 325,
          "lowShelfGain": 6,
          "peakFreq": 800,
          "peakGain": -3,
          "peakQ": 0.9,
          "highShelfFreq": 1900,
          "highShelfGain": -7,
          "lpfFreq": 3200
        },
        "cockpit": {
          "gain": 1,
          "lowShelfFreq": 200,
          "lowShelfGain": 0,
          "peakFreq": 1000,
          "peakGain": 0,
          "peakQ": 1,
          "highShelfFreq": 3000,
          "highShelfGain": 0,
          "lpfFreq": 20000
        },
        "exterior": {
          "gain": 1,
          "lowShelfFreq": 200,
          "lowShelfGain": 0,
          "peakFreq": 1000,
          "peakGain": 0,
          "peakQ": 1,
          "highShelfFreq": 3000,
          "highShelfGain": 0,
          "lpfFreq": 20000
        },
        "exteriorFront": {
          "gain": 1,
          "lowShelfFreq": 250,
          "lowShelfGain": -3,
          "peakFreq": 1400,
          "peakGain": 2,
          "peakQ": 1.1,
          "highShelfFreq": 2200,
          "highShelfGain": 3,
          "lpfFreq": 16000
        },
        "exteriorBehind": {
          "gain": 1,
          "lowShelfFreq": 250,
          "lowShelfGain": 4,
          "peakFreq": 900,
          "peakGain": -1.5,
          "peakQ": 0.9,
          "highShelfFreq": 2200,
          "highShelfGain": -6,
          "lpfFreq": 7000
        }
      },
      "views": {
        "cockpit": ["cockpit", "jumpseat", "jump seat", "virtual cockpit", "vc"],
        "wing2": ["wing 2", "wing2", "left wing", "right wing"],
        "side": ["left wing", "right wing", "engine"],
        "engine": ["cfm", "iaev", "v2500", "r", "l"],
        "hasWingEngine": true
      },
      "views2": {
        "cockpit": ["cockpit", "jumpseat", "jump seat", "virtual cockpit", "vc", "2d cockpit"],
        "wing2": ["wing 2", "wing2", "left wing", "right wing", "wing view", "external wing"],
        "side": [],
        "engine": [],
        "hasWingEngine": false
      },
      "mix": {
        "masterBoost": 1.8,
        "cockpitAtten": 0.4,
        "wing2Atten": 0.85,
        "preGainDb": 0,
        "compThresholdDb": -18,
        "compRatio": 2,
        "cockpitInteriorBlend": 0.95,
        "wing2InteriorBlend": 0.9,
        "exteriorInteriorBlend": 0,
        "wing2EngineAtten": 0.55,
        "wingEngineAtten": 0.65,
        "interiorSmoothTc": 0.025,
        "wing2VibeDepth": 0.015,
        "wing2VibeRate": 2.5,
        "exteriorAmbLevel": 0.2,
        "exteriorLpfFreq": 1800,
        "tireScreechVol": 0.45,
        "togaGain": 1.15,
        "engineLayerAtten": 0.36,
        "idleWing2Boost": 1
      },
      "engine": {
        "rpmMin": 1000,
        "rpmShutdown": 990,
        "startupSuppressMs": 2000,
        "spawnProtectionMs": 8000,
        "altitudeProtectionFt": 1000,
        "idleUnfiltered": false
      },
      "curve": {
        "idleCut": 0.45,
        "n1Start": 0.12,
        "n1Full": 0.52,
        "togaStart": 0.42,
        "midCenter": 0.33,
        "midWidth": 0.32,
        "midGain": 2,
        "floor": 0.1,
        "togaExp": 1.25,
        "bellExp": 1.6,
        "idleBell": 0.75,
        "n1Bell": "bell",
        "togaPow": 1.1,
        "idleFloorMul": 1.25,
        "idleFinalMul": 1,
        "n1FloorMul": 0.95,
        "togaBoost": 0.8,
        "togaFinalMul": null,
        "idleRateMul": 9,
        "idleRateTail": 0.9,
        "n1RateK": 0.22,
        "togaRateK": 0.5,
        "togaRateExp": 1.15,
        "idlePower": 0.5,
        "idlePitchIntensity": 0.75,
        "idleVolumeBoost": 2,
        "n1Scale": 0.35,
        "maxLayerGain": 1,
        "n1BellMode": "bell",
        "n1BellMul": 1
      },
      "buzzsaw": {
        "lpf": 3200,
        "altMaxFt": 8000,
        "altExp": 1.6,
        "altScale": 1.5,
        "throttleStart": 0.65,
        "throttleEnd": 1,
        "rateBase": 0.78,
        "rateSpan": 0.2,
        "wingEngineBoost": 1,
        "wing2Atten": 1,
        "cockpitAtten": 1
      },
      "toga": {
        "altVolume": null,
        "cockpitMult": 1,
        "altitudeHF": false
      },
      "gear": {
        "lpfFreq": 1400,
        "lpfQ": 0.6,
        "fadeIn": 0.4,
        "fadeOut": 0.4,
        "volCockpit": 0.633,
        "volWing2": 0.437,
        "volWingEngine": 0.518
      },
      "assets": {
        "ambWing2Air": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A320-CEO-Family-Sound-Pack-Official/main/freesound_community-airplane-interior-ambience-59644.mp3",
        "ambWing2": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/freesound_community-plane-interior-52034%20(1).mp3",
        "ambExterior": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Weather_%23.wav",
        "gearClick": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Master_9.wav",
        "spoiler": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-Sound-Pack-Official/main/Master_10.wav",
        "engineToggle": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Master_19.wav",
        "rain": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Airbus-A320-Sound-Pack/main/Weather_6.wav",
        "tire": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-Sound-Pack-Official/main/freesound_community-cinematic-deep-rumble-6418.mp3",
        "tireScreech": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Sound-Pack-Ultimate-/main/Tire%20Screech.wav",
        "rattle": {
          "low": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Dynamics_20.wav",
          "midL": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Dynamics_2.wav",
          "transC": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Dynamics_18.wav"
        },
        "startupExt": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B737NG-Family-Sound-Repository/main/B737Startup.wav",
        "startupInt": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Airbus-A320-Sound-Pack/main/A320%20star2_INN.wav",
        "shutdown": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Airbus-A320-Sound-Pack/main/A320%20Engine%20ShutOff.wav",
        "ambCockpit": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B737NG-Family-Sound-Pack/main/B737CockpitAmbience.wav",
        "buzzsaw": "",
        "gearHum": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-Sound-Pack-Official/main/Gear%20Down%20(3).wav",
        "tray": "",
        "flapClick": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Master_13.wav",
        "flapMotor": null,
        "flapHum": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Airbus-A320-Sound-Pack/main/Dynamics_5.wav",
        "configAlarm": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Sound-Pack-Ultimate-/main/B737CONFIG.wav"
      }
    },
    // ---- B737 MAX ------------------------------------------------------
    {
      "code": "b737max",
      "suffix": "B737MAX",
      "name": "B737 MAX",
      "ids": ["2772", "2769"],
      "layers": {
        "idle": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B737-MAX-Family-Repository/main/B737MAXIDLE.wav",
        "n1": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B737-MAX-Family-Repository/main/B737MAXN1.wav",
        "toga": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B737-MAX-Family-Repository/main/B737MAXTOGA.wav"
      },
      "layerLpf": {
        "idle": 4000,
        "n1": 4500,
        "toga": 6000
      },
      "engineLpf": {
        "default": {
          "idle": 4000,
          "n1": 4500,
          "toga": 6000,
          "buzzsaw": 3200
        },
        "interior": {
          "cockpit": {
            "idle": 1500,
            "n1": 1500,
            "toga": 800,
            "buzzsaw": 1500
          },
          "wing2": {
            "idle": 3200,
            "n1": 3400,
            "toga": 700,
            "buzzsaw": 1700
          },
          "wingEngine": {
            "idle": 4000,
            "n1": 4500,
            "toga": 6000,
            "buzzsaw": 3200
          }
        }
      },
      "profiles": {
        "wingEngine": {
          "gain": 0.65,
          "lowShelfFreq": 275,
          "lowShelfGain": 6,
          "peakFreq": 1000,
          "peakGain": -4,
          "peakQ": 1.3,
          "highShelfFreq": 2000,
          "highShelfGain": -6,
          "lpfFreq": 3800
        },
        "wing2": {
          "gain": 1,
          "lowShelfFreq": 325,
          "lowShelfGain": 6,
          "peakFreq": 800,
          "peakGain": -3,
          "peakQ": 0.9,
          "highShelfFreq": 1900,
          "highShelfGain": -7,
          "lpfFreq": 3200
        },
        "cockpit": {
          "gain": 1,
          "lowShelfFreq": 200,
          "lowShelfGain": 0,
          "peakFreq": 1000,
          "peakGain": 0,
          "peakQ": 1,
          "highShelfFreq": 3000,
          "highShelfGain": 0,
          "lpfFreq": 20000
        },
        "exterior": {
          "gain": 1,
          "lowShelfFreq": 200,
          "lowShelfGain": 0,
          "peakFreq": 1000,
          "peakGain": 0,
          "peakQ": 1,
          "highShelfFreq": 3000,
          "highShelfGain": 0,
          "lpfFreq": 20000
        },
        "exteriorFront": {
          "gain": 1,
          "lowShelfFreq": 250,
          "lowShelfGain": -3,
          "peakFreq": 1400,
          "peakGain": 2,
          "peakQ": 1.1,
          "highShelfFreq": 2200,
          "highShelfGain": 3,
          "lpfFreq": 16000
        },
        "exteriorBehind": {
          "gain": 1,
          "lowShelfFreq": 250,
          "lowShelfGain": 4,
          "peakFreq": 900,
          "peakGain": -1.5,
          "peakQ": 0.9,
          "highShelfFreq": 2200,
          "highShelfGain": -6,
          "lpfFreq": 7000
        }
      },
      "views": {
        "cockpit": ["cockpit", "jumpseat", "jump seat", "virtual cockpit", "vc"],
        "wing2": ["wing 2", "wing2", "left wing", "right wing"],
        "side": ["left wing", "right wing", "engine"],
        "engine": ["cfm", "iaev", "v2500", "r", "l"],
        "hasWingEngine": true
      },
      "views2": {
        "cockpit": ["cockpit", "jumpseat", "jump seat", "virtual cockpit", "vc", "2d cockpit"],
        "wing2": ["wing 2", "wing2", "left wing", "right wing", "wing view", "external wing"],
        "side": [],
        "engine": [],
        "hasWingEngine": false
      },
      "mix": {
        "masterBoost": 2.07,
        "cockpitAtten": 0.8,
        "wing2Atten": 1.445,
        "preGainDb": 0,
        "compThresholdDb": -18,
        "compRatio": 2,
        "cockpitInteriorBlend": 0.95,
        "wing2InteriorBlend": 0.9,
        "exteriorInteriorBlend": 0,
        "wing2EngineAtten": 0.55,
        "wingEngineAtten": 0.65,
        "interiorSmoothTc": 0.025,
        "wing2VibeDepth": 0.015,
        "wing2VibeRate": 2.5,
        "exteriorAmbLevel": 0.2,
        "exteriorLpfFreq": 1800,
        "tireScreechVol": 0.45,
        "togaGain": 1.035,
        "engineLayerAtten": 0.36,
        "idleWing2Boost": 1
      },
      "engine": {
        "rpmMin": 1000,
        "rpmShutdown": 990,
        "startupSuppressMs": 2000,
        "spawnProtectionMs": 8000,
        "altitudeProtectionFt": 1000,
        "idleUnfiltered": false
      },
      "curve": {
        "idleCut": 0.45,
        "n1Start": 0.12,
        "n1Full": 0.52,
        "togaStart": 0.42,
        "midCenter": 0.33,
        "midWidth": 0.32,
        "midGain": 2,
        "floor": 0.1,
        "togaExp": 1.25,
        "bellExp": 1.6,
        "idleBell": 0.75,
        "togaPow": 1.1,
        "idleFloorMul": 1.25,
        "idleFinalMul": 1,
        "n1FloorMul": 0.95,
        "togaBoost": 0.8,
        "togaFinalMul": null,
        "idleRateMul": 9,
        "idleRateTail": 0.9,
        "n1RateK": 0.22,
        "togaRateK": 0.5,
        "togaRateExp": 1.15,
        "idlePower": 0.5,
        "idlePitchIntensity": 0.4,
        "idleVolumeBoost": 2,
        "n1Scale": 0.385,
        "maxLayerGain": 1.1,
        "n1BellMode": "bell",
        "n1BellMul": 1.2
      },
      "buzzsaw": {
        "lpf": 3200,
        "altMaxFt": 8000,
        "altExp": 1.6,
        "altScale": 1.5,
        "throttleStart": 0.65,
        "throttleEnd": 1,
        "rateBase": 0.78,
        "rateSpan": 0.2,
        "wingEngineBoost": 1,
        "wing2Atten": 1,
        "cockpitAtten": 1
      },
      "toga": {
        "altVolume": null,
        "cockpitMult": 1,
        "altitudeHF": false
      },
      "gear": {
        "lpfFreq": 1400,
        "lpfQ": 0.6,
        "fadeIn": 0.4,
        "fadeOut": 0.4,
        "volCockpit": 0.633,
        "volWing2": 0.437,
        "volWingEngine": 0.518
      },
      "assets": {
        "ambWing2Air": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A320-CEO-Family-Sound-Pack-Official/main/freesound_community-airplane-interior-ambience-59644.mp3",
        "ambWing2": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/freesound_community-plane-interior-52034%20(1).mp3",
        "ambExterior": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Weather_%23.wav",
        "gearClick": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Master_9.wav",
        "spoiler": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-Sound-Pack-Official/main/Master_10.wav",
        "engineToggle": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Master_19.wav",
        "rain": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Airbus-A320-Sound-Pack/main/Weather_6.wav",
        "tire": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-Sound-Pack-Official/main/freesound_community-cinematic-deep-rumble-6418.mp3",
        "tireScreech": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Sound-Pack-Ultimate-/main/Tire%20Screech.wav",
        "rattle": {
          "low": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Dynamics_20.wav",
          "midL": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Dynamics_2.wav",
          "transC": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Dynamics_18.wav"
        },
        "startupExt": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B737-MAX-Family-Repository/main/B737MAXSTARTUP.wav",
        "startupInt": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B737-MAX-Family-Repository/main/engine_53.wav",
        "shutdown": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B737-MAX-Family-Repository/main/engine_28.wav",
        "ambCockpit": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B737NG-Family-Sound-Pack/main/B737CockpitAmbience.wav",
        "buzzsaw": "",
        "gearHum": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B737-MAX-Family-Repository/main/B737MAXGEARSOUND.wav",
        "tray": "",
        "flapClick": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B737-MAX-Family-Repository/main/FL1.wav",
        "flapMotor": null,
        "flapHum": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B737-MAX-Family-Repository/main/FlapsR.wav",
        "configAlarm": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Sound-Pack-Ultimate-/main/B737CONFIG.wav"
      }
    },
    // ---- B777 ----------------------------------------------------------
    {
      "code": "b777",
      "suffix": "B777",
      "name": "B777",
      "ids": ["25", "4402", "240", "1004"],
      "layers": {
        "idle": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-Sound-Repository/main/B777IDLE.wav",
        "n1": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-Sound-Repository/main/B777N1.wav",
        "toga": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-Sound-Repository/main/B777TOGA.wav"
      },
      "layerLpf": {
        "idle": 20000,
        "n1": 20000,
        "toga": 20000
      },
      "engineLpf": {
        "default": {
          "idle": 5300,
          "n1": 5300,
          "toga": 5300,
          "buzzsaw": 2800
        },
        "interior": {
          "cockpit": {
            "idle": 1350,
            "n1": 1350,
            "toga": 1350,
            "buzzsaw": 1350
          },
          "wing2": {
            "idle": 1800,
            "n1": 1800,
            "toga": 400,
            "buzzsaw": 700
          },
          "wingEngine": {
            "idle": 5300,
            "n1": 5300,
            "toga": 5300,
            "buzzsaw": 2800
          }
        }
      },
      "profiles": {
        "wingEngine": {
          "gain": 1,
          "lowShelfFreq": 200,
          "lowShelfGain": 0,
          "peakFreq": 1000,
          "peakGain": 0,
          "peakQ": 1,
          "highShelfFreq": 3000,
          "highShelfGain": 0,
          "lpfFreq": 20000
        },
        "wing2": {
          "gain": 1,
          "lowShelfFreq": 200,
          "lowShelfGain": 0,
          "peakFreq": 1000,
          "peakGain": 0,
          "peakQ": 1,
          "highShelfFreq": 3000,
          "highShelfGain": 0,
          "lpfFreq": 20000
        },
        "cockpit": {
          "gain": 1,
          "lowShelfFreq": 200,
          "lowShelfGain": 0,
          "peakFreq": 1000,
          "peakGain": 0,
          "peakQ": 1,
          "highShelfFreq": 3000,
          "highShelfGain": 0,
          "lpfFreq": 20000
        },
        "exterior": {
          "gain": 1,
          "lowShelfFreq": 200,
          "lowShelfGain": 0,
          "peakFreq": 1000,
          "peakGain": 0,
          "peakQ": 1,
          "highShelfFreq": 3000,
          "highShelfGain": 0,
          "lpfFreq": 20000
        },
        "exteriorFront": {
          "gain": 1,
          "lowShelfFreq": 250,
          "lowShelfGain": -3,
          "peakFreq": 1400,
          "peakGain": 2,
          "peakQ": 1.1,
          "highShelfFreq": 2200,
          "highShelfGain": 3,
          "lpfFreq": 16000
        },
        "exteriorBehind": {
          "gain": 1,
          "lowShelfFreq": 250,
          "lowShelfGain": 4,
          "peakFreq": 900,
          "peakGain": -1.5,
          "peakQ": 0.9,
          "highShelfFreq": 2200,
          "highShelfGain": -6,
          "lpfFreq": 7000
        }
      },
      "views": {
        "cockpit": ["cockpit", "jumpseat", "jump seat", "virtual cockpit", "vc"],
        "wing2": ["wing 2", "wing2", "left wing", "right wing"],
        "side": [],
        "engine": [],
        "hasWingEngine": false
      },
      "views2": {
        "cockpit": ["cockpit", "jumpseat", "jump seat", "virtual cockpit", "vc", "2d cockpit"],
        "wing2": ["wing 2", "wing2", "left wing", "right wing", "wing view", "external wing"],
        "side": [],
        "engine": [],
        "hasWingEngine": false
      },
      "mix": {
        "masterBoost": 2.2,
        "cockpitAtten": 0.52,
        "wing2Atten": 0.75,
        "preGainDb": 0,
        "compThresholdDb": -18,
        "compRatio": 2,
        "cockpitInteriorBlend": 0.75,
        "wing2InteriorBlend": 0.7,
        "exteriorInteriorBlend": 0,
        "wing2EngineAtten": 0.48,
        "wingEngineAtten": 0.55,
        "interiorSmoothTc": 0.025,
        "wing2VibeDepth": 0.015,
        "wing2VibeRate": 2.5,
        "exteriorAmbLevel": 0.2,
        "exteriorLpfFreq": 1800,
        "tireScreechVol": 0.45,
        "togaGain": 1.15,
        "engineLayerAtten": 1,
        "idleWing2Boost": 1
      },
      "engine": {
        "rpmMin": 1000,
        "rpmShutdown": 900,
        "startupSuppressMs": 2000,
        "spawnProtectionMs": 8000,
        "altitudeProtectionFt": 1000,
        "idleUnfiltered": false
      },
      "curve": {
        "idleCut": 0.48,
        "n1Start": 0.1105,
        "n1Full": 0.48,
        "togaStart": 0.7,
        "midCenter": 0.33,
        "midWidth": 0.32,
        "midGain": 2,
        "floor": 0.1,
        "togaExp": 1.15,
        "bellExp": 1.6,
        "idleBell": 0.45,
        "n1Bell": 0.95,
        "togaPow": 1,
        "idleFloorMul": 0.8,
        "idleFinalMul": 0.35,
        "n1FloorMul": 1.25,
        "togaBoost": 0.8,
        "togaFinalMul": 0.8,
        "idleRateMul": 9,
        "idleRateTail": 0.9,
        "n1RateK": 0.25,
        "togaRateK": 0.25,
        "togaRateExp": 1.05,
        "idlePower": 0.4,
        "idlePitchIntensity": 0.38,
        "idleVolumeBoost": 1.25,
        "n1Scale": 0.58,
        "maxLayerGain": 1.8,
        "n1BellMode": "const",
        "n1BellMul": 0.95
      },
      "buzzsaw": {
        "lpf": 2800,
        "altMaxFt": 8000,
        "altExp": 1.6,
        "altScale": 1.5,
        "throttleStart": 0.65,
        "throttleEnd": 1,
        "rateBase": 0.78,
        "rateSpan": 0.2,
        "wingEngineBoost": 1,
        "wing2Atten": 1,
        "cockpitAtten": 1
      },
      "toga": {
        "altVolume": {
          "floor": 0.11,
          "maxFt": 12000
        },
        "cockpitMult": 0.8,
        "altitudeHF": false
      },
      "gear": {
        "lpfFreq": 1400,
        "lpfQ": 0.6,
        "fadeIn": 0.4,
        "fadeOut": 0.4,
        "volCockpit": 0.633,
        "volWing2": 0.437,
        "volWingEngine": 0.748
      },
      "assets": {
        "ambWing2Air": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-A320-CEO-Family-Sound-Pack-Official/main/freesound_community-airplane-interior-ambience-59644.mp3",
        "ambWing2": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/freesound_community-plane-interior-52034%20(1).mp3",
        "ambExterior": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Weather_%23.wav",
        "gearClick": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Master_9.wav",
        "spoiler": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-Sound-Pack-Official/main/Master_10.wav",
        "engineToggle": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Master_19.wav",
        "rain": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Airbus-A320-Sound-Pack/main/Weather_6.wav",
        "tire": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-Sound-Pack-Official/main/freesound_community-cinematic-deep-rumble-6418.mp3",
        "tireScreech": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Sound-Pack-Ultimate-/main/Tire%20Screech.wav",
        "rattle": {
          "low": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Dynamics_20.wav",
          "midL": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Dynamics_2.wav",
          "transC": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Dynamics_18.wav"
        },
        "startupExt": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-Sound-Pack-Official/main/B777-300ER%20GE90%20Start%20Up%20Sound%20(2).mp3",
        "startupInt": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/GEEngine_12.mp3",
        "shutdown": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-Sound-Pack-Official/main/B777-300ER%20ENGINE%20SHUTOFF%20(1).mp3",
        "ambCockpit": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Master_14.wav",
        "buzzsaw": "",
        "gearHum": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-Sound-Pack-Official/main/Gear%20Down%20(3).wav",
        "tray": "",
        "flapClick": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-300ER-Sound-Pack-Official/main/Master_13.wav",
        "flapMotor": null,
        "flapHum": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-B777-Sound-Repository/main/B777FLAP.wav",
        "configAlarm": "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Sound-Pack-Ultimate-/main/B777CONFIG.wav"
      }
    }
  ];
  GEFMOD_BANKS.forEach(B => { try { createPack(B); } catch(e){ console.warn('[GEFMOD] bank failed', B.code, e); } });
  // SHARED EMERGENCY MUTE/UNMUTE: dispatches to whichever pack is currently active, on top of each module's own emergencyMute_<CODE>/emergencyUnmute_<CODE>.
  window.emergencyMute = function(){
    const suffixes = ['A320NEO','A320','A330','A339','A380','A340','A350','B737','B737MAX','B777'];
    for (const s of suffixes) {
      const fn = window['emergencyMute_' + s];
      if (window._GEPacks && window._GEPacks[s.toLowerCase()] && typeof fn === 'function') {
        return fn();
      }
    }
  };
  window.emergencyUnmute = function(){
    const suffixes = ['A320NEO','A320','A330','A339','A380','A340','A350','B737','B737MAX','B777'];
    for (const s of suffixes) {
      const fn = window['emergencyUnmute_' + s];
      if (window._GEPacks && window._GEPacks[s.toLowerCase()] && typeof fn === 'function') {
        return fn();
      }
    }
  };
  // SHARED VOLUME SLIDER: one UI shared by all six packs, always targeting whichever pack is active via window._GEPacks.
  (function(){
    const PACK_INFO = [
      { code: 'a320neo', prefix: 'GE90A320NEO', label: 'A320neo LEAP 1A/PW 1100G Volume' },
      { code: 'a320', prefix: 'GE90A320', label: 'A320 CFM56/IAEV2500 Volume' },
      { code: 'a330', prefix: 'GE90A330', label: 'A330 Trent 700/CF6 Volume'  },
      { code: 'a339', prefix: 'GE90A339', label: 'A330-900neo Trent 7000 Volume' },
      { code: 'a380', prefix: 'GE90A380', label: 'A380 Trent 900/GP7200 Volume' },
      { code: 'a340', prefix: 'GE90A340', label: 'A340 CFM56-5C/Trent 500 Volume' },
      { code: 'a350', prefix: 'GE90A350', label: 'A350 Trent XWB Volume'     },
      { code: 'b737', prefix: 'GE90B737', label: 'B737 CFM56 Volume'},
      { code: 'b737max', prefix: 'GE90B737MAX', label: 'B737 MAX CFM LEAP-1B Volume' },
      { code: 'b777', prefix: 'GE90B777', label: 'B777 GE90 Volume'          },
    ];
    function activePackInfo(){
      for (const p of PACK_INFO) {
        if (window._GEPacks && window._GEPacks[p.code]) return p;
      }
      return null;
    }
    // Internal volume variable (0-1), shared across all packs
    window._GE_customVolume = window._GE_customVolume ?? 1.0;
    window._GE_sliderVisible = window._GE_sliderVisible ?? true;
    // Persisted 'hide panel' hotkey, stored as a KeyboardEvent.code value (Shift-independent). Defaults to '.'.
    function loadHideKey(){
      try {
        const saved = localStorage.getItem('GE90_hideKey');
        if (saved) return saved;
      } catch(e){}
      return 'Period';
    }
    window._GE_hideKey = window._GE_hideKey ?? loadHideKey();
    window._GE_listeningForHideKey = false;
    function prettyKeyLabel(code){
      if (!code) return '(none)';
      if (code.startsWith('Key')) return code.slice(3);
      if (code.startsWith('Digit')) return code.slice(5);
      const specialNames = {
        'Period': '.', 'Comma': ',', 'Slash': '/', 'Backquote': '`',
        'Semicolon': ';', 'Quote': "'", 'BracketLeft': '[', 'BracketRight': ']',
        'Backslash': '\\', 'Minus': '-', 'Equal': '=', 'Space': 'Space',
        'Escape': 'Esc', 'Backspace': 'Backspace', 'Tab': 'Tab', 'Enter': 'Enter'
      };
      return specialNames[code] || code;
    }
    function loadGpwsOn(){
      try {
        const saved = localStorage.getItem('GE90_gpwsOn');
        if (saved !== null) return saved === 'true';
      } catch(e){}
      return true;
    }
    window._GE_gpwsOn = window._GE_gpwsOn ?? loadGpwsOn();
    setInterval(function(){ window.soundsOn = !!window._GE_gpwsOn && !window._GE_gpwsSKeyMuted; }, 250);
    function loadSeatbeltOn(){
      try {
        const saved = localStorage.getItem('GE90_seatbeltOn');
        if (saved !== null) return saved === 'true';
      } catch(e){}
      return false;
    }
    window._GE_seatbeltOn = window._GE_seatbeltOn ?? loadSeatbeltOn();
    const SEATBELT_SOUND_URL =
      'https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Seatbelt-Sign-Sound/main/670297__kinoton__airplane-seatbelt-sign-beep.mp3';
    window._GE_seatbeltChime = window._GE_seatbeltChime || new Audio(SEATBELT_SOUND_URL);
    window._GE_seatbeltChime.dataset.ge90Allow = 'true';
    window._GE_seatbeltChime.preload = 'auto';
    window._GE_seatbeltChime.volume = 0.85;
    function playSeatbeltChime(){
      try {
        const chime = window._GE_seatbeltChime;
        chime.pause();
        chime.currentTime = 0;
        const p = chime.play();
        if (p && typeof p.catch === 'function') {
          p.catch(err => console.warn('[GE90 Ultimate] seatbelt chime blocked by browser', err));
        }
      } catch(err) {
        console.warn('[GE90 Ultimate] seatbelt chime error', err);
      }
    }
    function loadAcOn(){
      try {
        const saved = localStorage.getItem('GE90_acOn');
        if (saved !== null) return saved === 'true';
      } catch(e){}
      return false;
    }
    window._GE_acOn = window._GE_acOn ?? loadAcOn();
    const AC_SOUND_URL =
      'https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Sound-Pack-Ultimate-/main/AirConditionerSound.wav';
    const AC_TARGET_VOLUME = 0.7;
    const AC_TARGET_VOLUME_BY_PACK = {
      a340: 0.5,
      a320neo: 0.5
    };
    function _GE_acTargetVolume(){
      const info = activePackInfo();
      let base = (info && Object.prototype.hasOwnProperty.call(AC_TARGET_VOLUME_BY_PACK, info.code))
        ? AC_TARGET_VOLUME_BY_PACK[info.code]
        : AC_TARGET_VOLUME;
      // A330 CEO: wing2 (WingL2/R2) and wingEngine (WingL1/R1) views get 30% quieter AC
      if (info && info.code === 'a330') {
        const v = _GE_acGetViewType();
        if (v === 'wing2' || v === 'wingEngine') base *= 0.7;
      }
      return base;
    }
    const AC_VIEWS = ['cockpit', 'wing2', 'wingEngine'];
    const AC_CROSSFADE_SEC = 1.5;
    function _GE_makeAcAudio(){
      const a = new Audio(AC_SOUND_URL);
      a.loop = false;
      a.dataset.ge90Allow = 'true';
      a.preload = 'auto';
      a.volume = 0;
      return a;
    }
    window._GE_acSoundA = window._GE_acSoundA || _GE_makeAcAudio();
    window._GE_acSoundB = window._GE_acSoundB || _GE_makeAcAudio();
    window._GE_acActiveSlot = window._GE_acActiveSlot || 'A';
    window._GE_acCrossfading = window._GE_acCrossfading || false;
    window._GE_acCrossfadeStart = 0;
    window._GE_acVolume = window._GE_acVolume || 0;
    function _GE_acFrontBack(){
      return window._GE_acActiveSlot === 'A'
        ? { front: window._GE_acSoundA, back: window._GE_acSoundB }
        : { front: window._GE_acSoundB, back: window._GE_acSoundA };
    }
    function _GE_acGetViewType(){
      try {
        const cam = window.geofs?.camera;
        if (!cam) return "exterior";
        let s = cam.currentModeName || cam.currentView || cam.currentDefinition?.name || "";
        s = String(s).toLowerCase();
        if (!s) return "exterior";
        if (s.includes("cockpit") || s.includes("jumpseat") || s.includes("jump seat") ||
            s.includes("virtual cockpit") || s.includes("vc")) return "cockpit";
        // A330 CEO names its camera modes "WingR1/WingL1" (close engine view)
        // and "WingR2/WingL2" (cabin/window view) instead of the generic
        // "left wing"/"right wing" strings other aircraft use. Shared by AC
        // and APU classifiers — without this, both wing views fell through to
        // "exterior", so the APU sounded the same inside and outside.
        if (s.includes("wingr1") || s.includes("wingl1") || s.includes("business class")) return "wingEngine";
        if (s.includes("wingr2") || s.includes("wingl2")) return "wing2";
        if ((s.includes("left wing") || s.includes("right wing") || s.includes("right") || s.includes("left")) &&
            (s.includes("cfm") || s.includes("iaev") || s.includes("v2500") || s.includes("engine"))) return "wingEngine";
        if (s.includes("wing 2") || s.includes("wing2") || s.includes("left wing") || s.includes("right wing")) return "wing2";
        return "exterior";
      } catch(e){
        return "exterior";
      }
    }
    function _GE_acShouldPlay(){
      try {
        if (!activePackInfo()) return false;
        const sKeyMuted = !!window._GE_gpwsSKeyMuted;
        const simPaused = !!(window.geofs && typeof window.geofs.isPaused === 'function' && window.geofs.isPaused());
        const inAcView = AC_VIEWS.includes(_GE_acGetViewType());
        return !!window._GE_acOn && inAcView && !sKeyMuted && !simPaused;
      } catch(e){
        return false;
      }
    }
    function _GE_acTick(){
      try {
        const target = _GE_acShouldPlay() ? _GE_acTargetVolume() : 0;
        window._GE_acVolume += (target - window._GE_acVolume) * 0.15;
        if (Math.abs(window._GE_acVolume - target) < 0.004) window._GE_acVolume = target;
        const master = window._GE_acVolume;
        const { front, back } = _GE_acFrontBack();
        if (master <= 0.0005) {
          if (!front.paused) { try { front.pause(); } catch(e){} }
          if (!back.paused) { try { back.pause(); } catch(e){} }
          front.volume = 0;
          back.volume = 0;
          window._GE_acCrossfading = false;
          return;
        }
        if (front.paused && !window._GE_acCrossfading) {
          front.currentTime = 0;
          const p = front.play();
          if (p && typeof p.catch === 'function') p.catch(()=>{});
        }
        if (!window._GE_acCrossfading) {
          front.volume = master;
          back.volume = 0;
          if (!back.paused) { try { back.pause(); } catch(e){} }
          const dur = front.duration;
          if (dur && !isNaN(dur) && dur > AC_CROSSFADE_SEC * 2 && !front.paused &&
              front.currentTime >= dur - AC_CROSSFADE_SEC) {
            back.currentTime = 0;
            const p = back.play();
            if (p && typeof p.catch === 'function') p.catch(()=>{});
            window._GE_acCrossfading = true;
            window._GE_acCrossfadeStart = performance.now();
          }
        } else {
          const elapsed = (performance.now() - window._GE_acCrossfadeStart) / 1000;
          const t = Math.min(1, elapsed / AC_CROSSFADE_SEC);
          const angle = t * Math.PI / 2;
          front.volume = master * Math.cos(angle);
          back.volume = master * Math.sin(angle);
          if (t >= 1) {
            try { front.pause(); front.currentTime = 0; } catch(e){}
            window._GE_acActiveSlot = window._GE_acActiveSlot === 'A' ? 'B' : 'A';
            window._GE_acCrossfading = false;
          }
        }
      } catch(e){}
    }
    setInterval(_GE_acTick, 50);
    if (window._GE_acOn) {
      try {
        const p = _GE_acFrontBack().front.play();
        if (p && typeof p.catch === 'function') p.catch(()=>{});
      } catch(e){}
    }
    function createVolumeSlider(){
      if (document.getElementById('ge90-volume-wrap')) return;
      const wrap = document.createElement('div');
      wrap.id = 'ge90-volume-wrap';
      wrap.style.position = 'fixed';
      wrap.style.top = '110px';
      wrap.style.left = '10px';
      wrap.style.padding = '12px 16px';
      wrap.style.background = 'rgba(0,0,0,0.55)';
      wrap.style.borderRadius = '10px';
      wrap.style.backdropFilter = 'blur(6px)';
      wrap.style.boxShadow = '0 4px 12px rgba(0,0,0,0.35)';
      wrap.style.zIndex = '999999';
      wrap.style.opacity = '0.9';
      const label = document.createElement('div');
      label.id = 'ge90-volume-label';
      label.textContent = 'Engine Volume';
      label.style.color = 'white';
      label.style.fontSize = '13px';
      label.style.fontFamily = 'Arial, sans-serif';
      label.style.marginBottom = '6px';
      label.style.textAlign = 'center';
      const slider = document.createElement('input');
      slider.type = 'range';
      slider.min = '0';
      slider.max = '100';
      slider.value = String(Math.round(window._GE_customVolume * 100));
      slider.id = 'ge90-volume-slider';
      slider.style.width = '180px';
      slider.style.display = 'block';
      slider.style.cursor = 'pointer';
      slider.style.accentColor = '#4da3ff';
      slider.addEventListener('input', () => {
        window._GE_customVolume = Number(slider.value) / 100;
      });
      wrap.appendChild(label);
      wrap.appendChild(slider);
      const divider1 = document.createElement('div');
      divider1.id = 'ge90-volume-divider';
      divider1.style.borderTop = '1px solid rgba(255,255,255,0.2)';
      divider1.style.margin = '10px 0 8px 0';
      wrap.appendChild(divider1);
      const hideKeyRow = document.createElement('div');
      hideKeyRow.style.display = 'flex';
      hideKeyRow.style.alignItems = 'center';
      hideKeyRow.style.justifyContent = 'space-between';
      hideKeyRow.style.marginBottom = '8px';
      hideKeyRow.style.gap = '10px';
      const hideKeyLabel = document.createElement('div');
      hideKeyLabel.textContent = 'Hide Panel Key:';
      hideKeyLabel.style.color = 'white';
      hideKeyLabel.style.fontSize = '12px';
      hideKeyLabel.style.fontFamily = 'Arial, sans-serif';
      const hideKeyBtn = document.createElement('button');
      hideKeyBtn.id = 'ge90-hidekey-btn';
      hideKeyBtn.type = 'button';
      hideKeyBtn.textContent = prettyKeyLabel(window._GE_hideKey);
      hideKeyBtn.title = 'Click, then press any key to change the hide-panel shortcut';
      hideKeyBtn.style.fontSize = '12px';
      hideKeyBtn.style.fontFamily = 'Arial, sans-serif';
      hideKeyBtn.style.padding = '3px 10px';
      hideKeyBtn.style.borderRadius = '6px';
      hideKeyBtn.style.border = '1px solid rgba(255,255,255,0.3)';
      hideKeyBtn.style.background = 'rgba(255,255,255,0.12)';
      hideKeyBtn.style.color = 'white';
      hideKeyBtn.style.cursor = 'pointer';
      hideKeyBtn.style.minWidth = '64px';
      hideKeyBtn.addEventListener('click', () => {
        if (window._GE_listeningForHideKey) return;
        window._GE_listeningForHideKey = true;
        hideKeyBtn.textContent = 'Press a key…';
        hideKeyBtn.style.background = 'rgba(77,163,255,0.35)';
      });
      hideKeyRow.appendChild(hideKeyLabel);
      hideKeyRow.appendChild(hideKeyBtn);
      wrap.appendChild(hideKeyRow);
      const gpwsRow = document.createElement('div');
      gpwsRow.style.display = 'flex';
      gpwsRow.style.alignItems = 'center';
      gpwsRow.style.justifyContent = 'space-between';
      gpwsRow.style.gap = '10px';
      const gpwsLabel = document.createElement('div');
      gpwsLabel.textContent = 'GPWS Callouts:';
      gpwsLabel.style.color = 'white';
      gpwsLabel.style.fontSize = '12px';
      gpwsLabel.style.fontFamily = 'Arial, sans-serif';
      const gpwsBtn = document.createElement('button');
      gpwsBtn.id = 'ge90-gpws-toggle';
      gpwsBtn.type = 'button';
      gpwsBtn.style.fontSize = '12px';
      gpwsBtn.style.fontFamily = 'Arial, sans-serif';
      gpwsBtn.style.fontWeight = 'bold';
      gpwsBtn.style.padding = '3px 10px';
      gpwsBtn.style.borderRadius = '6px';
      gpwsBtn.style.border = '1px solid rgba(255,255,255,0.3)';
      gpwsBtn.style.cursor = 'pointer';
      gpwsBtn.style.minWidth = '64px';
      function refreshGpwsBtn(){
        if (window._GE_gpwsOn) {
          gpwsBtn.textContent = 'ON';
          gpwsBtn.style.background = 'rgba(70,200,120,0.35)';
          gpwsBtn.style.color = '#8fffb0';
        } else {
          gpwsBtn.textContent = 'OFF';
          gpwsBtn.style.background = 'rgba(220,70,70,0.35)';
          gpwsBtn.style.color = '#ff9d9d';
        }
      }
      refreshGpwsBtn();
   gpwsBtn.addEventListener('click', () => {
    window._GE_gpwsOn = !window._GE_gpwsOn;
    window.soundsOn = window._GE_gpwsOn && !window._GE_gpwsSKeyMuted;
    if (!window._GE_gpwsOn) {
        if (typeof window._GE_forceStopGpwsAudio === 'function') {
            window._GE_forceStopGpwsAudio();
        } else {
            const gpwsAudios = [
                window.a2500, window.a2000, window.a1000, window.a500, window.a400,
                window.a300, window.a200, window.a100, window.a50, window.a40,
                window.a30, window.a20, window.a10, window.aRetard, window.a5,
                window.stall, window.glideSlope, window.tooLowFlaps, window.tooLowGear,
                window.apDisconnect, window.minimumBaro, window.dontSink,
                window.masterA, window.bankAngle, window.overspeed, window.v1Callout
            ];
            gpwsAudios.forEach(a => {
                if (a) {
                    a.pause();
                    a.currentTime = 0;
                }
            });
        }
    }
    try { localStorage.setItem('GE90_gpwsOn', String(window._GE_gpwsOn)); } catch(e){}
    refreshGpwsBtn();
});
      gpwsRow.appendChild(gpwsLabel);
      gpwsRow.appendChild(gpwsBtn);
      wrap.appendChild(gpwsRow);
      const seatbeltRow = document.createElement('div');
      seatbeltRow.style.display = 'flex';
      seatbeltRow.style.alignItems = 'center';
      seatbeltRow.style.justifyContent = 'space-between';
      seatbeltRow.style.gap = '10px';
      seatbeltRow.style.margin = '10px 0 0 0';
      const seatbeltLabel = document.createElement('div');
      seatbeltLabel.textContent = 'Fasten Seatbelt:';
      seatbeltLabel.style.color = 'white';
      seatbeltLabel.style.fontSize = '12px';
      seatbeltLabel.style.fontFamily = 'Arial, sans-serif';
      const seatbeltBtn = document.createElement('button');
      seatbeltBtn.id = 'ge90-seatbelt-toggle';
      seatbeltBtn.type = 'button';
      seatbeltBtn.style.fontSize = '12px';
      seatbeltBtn.style.fontFamily = 'Arial, sans-serif';
      seatbeltBtn.style.fontWeight = 'bold';
      seatbeltBtn.style.padding = '3px 10px';
      seatbeltBtn.style.borderRadius = '6px';
      seatbeltBtn.style.border = '1px solid rgba(255,255,255,0.3)';
      seatbeltBtn.style.cursor = 'pointer';
      seatbeltBtn.style.minWidth = '64px';
      function refreshSeatbeltBtn(){
        if (window._GE_seatbeltOn) {
          seatbeltBtn.textContent = 'ON';
          seatbeltBtn.style.background = 'rgba(70,200,120,0.35)';
          seatbeltBtn.style.color = '#8fffb0';
        } else {
          seatbeltBtn.textContent = 'OFF';
          seatbeltBtn.style.background = 'rgba(220,70,70,0.35)';
          seatbeltBtn.style.color = '#ff9d9d';
        }
      }
      refreshSeatbeltBtn();
      seatbeltBtn.addEventListener('click', () => {
        window._GE_seatbeltOn = !window._GE_seatbeltOn;
        try { localStorage.setItem('GE90_seatbeltOn', String(window._GE_seatbeltOn)); } catch(e){}
        refreshSeatbeltBtn();
        playSeatbeltChime();
      });
      seatbeltRow.appendChild(seatbeltLabel);
      seatbeltRow.appendChild(seatbeltBtn);
      wrap.appendChild(seatbeltRow);
      const acRow = document.createElement('div');
      acRow.id = 'ge90-ac-row';
      acRow.style.display = 'flex';
      acRow.style.alignItems = 'center';
      acRow.style.justifyContent = 'space-between';
      acRow.style.gap = '10px';
      acRow.style.margin = '10px 0 0 0';
      const acLabel = document.createElement('div');
      acLabel.textContent = 'Air Conditioning:';
      acLabel.style.color = 'white';
      acLabel.style.fontSize = '12px';
      acLabel.style.fontFamily = 'Arial, sans-serif';
      const acBtn = document.createElement('button');
      acBtn.id = 'ge90-ac-toggle';
      acBtn.type = 'button';
      acBtn.style.fontSize = '12px';
      acBtn.style.fontFamily = 'Arial, sans-serif';
      acBtn.style.fontWeight = 'bold';
      acBtn.style.padding = '3px 10px';
      acBtn.style.borderRadius = '6px';
      acBtn.style.border = '1px solid rgba(255,255,255,0.3)';
      acBtn.style.cursor = 'pointer';
      acBtn.style.minWidth = '64px';
      function refreshAcBtn(){
        if (window._GE_acOn) {
          acBtn.textContent = 'ON';
          acBtn.style.background = 'rgba(70,200,120,0.35)';
          acBtn.style.color = '#8fffb0';
        } else {
          acBtn.textContent = 'OFF';
          acBtn.style.background = 'rgba(220,70,70,0.35)';
          acBtn.style.color = '#ff9d9d';
        }
      }
      refreshAcBtn();
      acBtn.addEventListener('click', () => {
        window._GE_acOn = !window._GE_acOn;
        try { localStorage.setItem('GE90_acOn', String(window._GE_acOn)); } catch(e){}
        refreshAcBtn();
        if (window._GE_acOn) {
          const p = _GE_acFrontBack().front.play();
          if (p && typeof p.catch === 'function') {
            p.catch(err => console.warn('[GE90 Ultimate] AC sound blocked by browser', err));
          }
        }
      });
      acRow.appendChild(acLabel);
      acRow.appendChild(acBtn);
      wrap.appendChild(acRow);
      // APU (Auxiliary Power Unit)
      // 4-state machine: off -> starting -> on -> shutting -> off. The two
      // transitional states are driven by the startup/shutdown clips' 'ended'
      // event, so the button can't be re-toggled mid-transition.
      function loadApuOn(){
        try {
          const saved = localStorage.getItem('GE90_apuOn');
          if (saved !== null) return saved === 'true';
        } catch(e){}
        return false;
      }
      window._GE_apuOn = window._GE_apuOn ?? loadApuOn();
      window._GE_apuState = window._GE_apuState || (window._GE_apuOn ? 'on' : 'off');
      const APU_STARTUP_URL =
        'https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Sound-Pack-Ultimate-/main/APUStartup.wav';
      const APU_LOOP_URL =
        'https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Sound-Pack-Ultimate-/main/APU.wav';
      const APU_SHUTDOWN_URL =
        'https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Sound-Pack-Ultimate-/main/APUShutdown.wav';
      // --- Web Audio graph -------------------------------------------------
      // Decoded buffers (not <audio> elements) so the loop can share
      // window._GE_seamlessLoopBuffer()'s crossfade, same as the ambience
      // beds/engine layers. One gain->lowpass->master chain driven per-frame
      // by camera-distance volume/muffling. No panning or Doppler.
      function loadApuAudio(){
        if (window._GE_apuAudioReady || window._GE_apuAudioLoading) return;
        window._GE_apuAudioLoading = true;
        try {
          const AC = _ORIG.AudioContext || window.AudioContext || window.webkitAudioContext;
          if (!AC) { window._GE_apuAudioLoading = false; return; }
          const actx = window._GE_apuCtx || (window._GE_apuCtx = new AC());
          actx._GE90_owned = true;
          const master = actx.createGain();
          master.gain.setValueAtTime(1, actx.currentTime);
          master._GE90_TAG = true;
          master.connect(actx.destination);
          // View lowpass: extra muffling specifically for cockpit/wing2 (as
          // if heard through the cabin/fuselage), left wide open for
          // wingEngine/exterior. Sits downstream of the distance lowpass.
          const interiorLpf = actx.createBiquadFilter();
          interiorLpf.type = 'lowpass';
          interiorLpf.frequency.value = 20000;
          interiorLpf.Q.value = 0.7;
          interiorLpf._GE90_TAG = true;
          interiorLpf.connect(master);
          const lpf = actx.createBiquadFilter();
          lpf.type = 'lowpass';
          lpf.frequency.value = 20000;
          lpf.Q.value = 0.7;
          lpf._GE90_TAG = true;
          lpf.connect(interiorLpf);
          const gain = actx.createGain();
          gain.gain.setValueAtTime(0, actx.currentTime);
          gain._GE90_TAG = true;
          gain.connect(lpf);
          window._GE_apuMaster = master;
          window._GE_apuLpf = lpf;
          window._GE_apuInteriorLpf = interiorLpf;
          window._GE_apuGain = gain;
          async function fetchDecode(url){
            const r = await fetch(url, { mode: 'cors' });
            const ab = await r.arrayBuffer();
            return await actx.decodeAudioData(ab);
          }
          Promise.all([
            fetchDecode(APU_STARTUP_URL),
            fetchDecode(APU_LOOP_URL),
            fetchDecode(APU_SHUTDOWN_URL)
          ]).then(([startupBuf, loopBuf, shutdownBuf]) => {
            window._GE_apuBuffers = {
              startup: startupBuf,
              loop: window._GE_seamlessLoopBuffer(actx, loopBuf, 0.6),
              shutdown: shutdownBuf
            };
            window._GE_apuAudioReady = true;
            window._GE_apuAudioLoading = false;
            if (typeof window._GE_apuOnBuffersReady === 'function') {
              const cb = window._GE_apuOnBuffersReady;
              window._GE_apuOnBuffersReady = null;
              cb();
            }
          }).catch(err => {
            window._GE_apuAudioLoading = false;
            console.warn('[GE90 Ultimate] APU audio load failed', err);
          });
        } catch(err) {
          window._GE_apuAudioLoading = false;
          console.warn('[GE90 Ultimate] APU audio setup failed', err);
        }
      }
      loadApuAudio();
      // Plays one cached buffer through the shared chain; tracks it as the
      // "active" source so the spatial tick below can Doppler-shift
      // whichever clip (startup/loop/shutdown) currently happens to be
      // playing.
      function _GE_apuPlayBuffer(buf, loop, onEnded){
        try {
          const actx = window._GE_apuCtx;
          const gain = window._GE_apuGain;
          if (!actx || !gain || !buf) return null;
          const src = actx.createBufferSource();
          src.buffer = buf;
          src.loop = !!loop;
          src._GE90_TAG = true;
          src.connect(gain);
          src.onended = onEnded || null;
          src.start(0);
          window._GE_apuActiveSrc = src;
          return src;
        } catch(err) {
          console.warn('[GE90 Ultimate] APU sound error', err);
          return null;
        }
      }
      // --- Camera-distance falloff --------------------------------------
      // Views grouped as if heard through the fuselage/cabin (quiet, extra
      // muffled) vs. everything that's acoustically "outside" the aircraft.
      const APU_VOLUME_QUIET_VIEWS = ['cockpit', 'wing2', 'wingEngine'];
      const APU_VOLUME_QUIET = 0.243;   // cockpit/wing2 (was 0.27, lowered 10% across all packs)
      const APU_VOLUME_OUTSIDE = 0.2907; // exterior (was 0.323, lowered 10% across all packs)
      const APU_INTERIOR_LPF_HZ = 1000;   // extra muffling cutoff for cockpit
      const APU_WING2_LPF_HZ = 700;       // slightly stronger muffling for wing2/WingL2/WingR2 (cabin window view)
      const APU_WINGL1R1_LPF_HZ = 500;    // stronger muffling for WingL1/WingR1 (close engine view, inside fuselage line)
      const APU_EXTERIOR_LPF_HZ = 20000; // effectively unfiltered for exterior
      const APU_DIST_NEAR = 3;        // meters — inside this, full volume/no extra muffling
      const APU_DIST_FAR  = 400;      // meters — heavily attenuated/muffled beyond this
      const APU_LPF_NEAR_HZ = 20000;
      const APU_LPF_FAR_HZ  = 1200;
      const APU_FALLOFF_FAR_MULT = 0.15; // still-audible floor at APU_DIST_FAR
      // The APU itself lives in the tail cone, not at the aircraft's main
      // position reference — so the sound source used for the distance
      // falloff is nudged this many meters behind that reference point,
      // along the reciprocal of the aircraft's heading.
      const APU_TAIL_OFFSET_M = 15;
      // Per-aircraft APU volume overrides — the APU sound system above is
      // shared/global (one instance for every plane), so without this check
      // any volume change here would apply to every aircraft, not just one.
      // IDs match the A330 CEO entry in the PACKS list ('a330').
      const APU_A330_IDS = ['244', '2856', '6012'];
      const APU_A330_STARTUP_MULT  = 0.9; // 10% quieter
      const APU_A330_LOOP_MULT     = 0.9; // 10% quieter
      const APU_A330_SHUTDOWN_MULT = 0.8; // 20% quieter
      // Exterior-view-only APU attenuation, by aircraft. Applies flatly across
      // startup/loop/shutdown (unlike the A330 per-state multipliers above).
      // Only one A340 bank ships (no separate -500/600), so its ids cover the
      // whole A340 entry.
      const APU_EXTERIOR_ATTEN_GROUPS = [
        { mult: 0.90,  ids: ['25','4402','240','1004'] },        // B777 — 10% quieter
        { mult: 0.90,  ids: ['24','2973','239'] },                // A350 — 10% quieter
        { mult: 0.80,  ids: ['10'] },                              // A380 — 20% quieter
        { mult: 0.75,  ids: ['5847','2871','2865','242','4646'] }, // A320neo — 25% quieter
        { mult: 0.80,  ids: ['5156','2879','3534','3011','5086','2870'] }, // A320 CEO — 20% quieter
        { mult: 0.80,  ids: ['244','2856','6012'] },               // A330 CEO — 20% quieter
        { mult: 0.85,  ids: ['4631'] },                            // A330neo — 15% quieter
        { mult: 0.775, ids: ['6006','2153','5998'] },              // A340-500/600 — 22.5% quieter
        { mult: 0.70,  ids: ['4','3054','5203','1001'] },          // B737NG — 30% quieter
        { mult: 0.70,  ids: ['2772','2769'] }                      // B737MAX — 30% quieter
      ];
      function _GE_apuExteriorAttenMult(acId){
        for (const g of APU_EXTERIOR_ATTEN_GROUPS) {
          if (g.ids.indexOf(acId) !== -1) return g.mult;
        }
        return 1;
      }
      function _GE_apuLlaToEcef(lat, lon, alt){
        const a = 6378137;
        const e2 = 0.00669437999014;
        const latR = lat * Math.PI/180;
        const lonR = lon * Math.PI/180;
        const N = a / Math.sqrt(1 - e2 * Math.sin(latR)**2);
        const x = (N + alt) * Math.cos(latR) * Math.cos(lonR);
        const y = (N + alt) * Math.cos(latR) * Math.sin(lonR);
        const z = (N * (1-e2) + alt) * Math.sin(latR);
        return { x, y, z };
      }
      function _GE_apuSpatialTick(){
        try {
          const actx = window._GE_apuCtx;
          const gain = window._GE_apuGain;
          const lpf = window._GE_apuLpf;
          const interiorLpf = window._GE_apuInteriorLpf;
          if (!actx || !gain || !lpf) return;
          const now = actx.currentTime;
          // Cockpit + wing2 get the much quieter, extra-muffled treatment;
          // wingEngine/exterior count as "outside".
          const view = _GE_acGetViewType();
          const isQuietView = APU_VOLUME_QUIET_VIEWS.includes(view);
          const baseVol = isQuietView ? APU_VOLUME_QUIET : APU_VOLUME_OUTSIDE;
          // WingL1/WingR1 (wingEngine) get the strongest APU muffle — right next to the engine.
          // WingL2/WingR2 (wing2) get a slightly stronger muffle than the cockpit (cabin window view).
          // Cockpit gets the standard interior cutoff; everything else is effectively unfiltered.
          const interiorTargetHz = (view === 'wingEngine') ? APU_WINGL1R1_LPF_HZ
                                 : (view === 'wing2')      ? APU_WING2_LPF_HZ
                                 : isQuietView             ? APU_INTERIOR_LPF_HZ
                                 :                           APU_EXTERIOR_LPF_HZ;
          let distMult = 1, lpfHz = APU_LPF_NEAR_HZ;
          const camPos = window.geofs?.camera?.cam?.position;
          const lla = window.geofs?.aircraft?.instance?.llaLocation;
          const heading = window.geofs?.animation?.values?.heading360;
          if (camPos && lla) {
            const latR = lla[0] * Math.PI/180;
            const lonR = lla[1] * Math.PI/180;
            const acEcef = _GE_apuLlaToEcef(lla[0], lla[1], lla[2]);
            // Local East/North unit vectors at the aircraft's position, used
            // to push the source back toward the tail.
            const eastX = -Math.sin(lonR), eastY = Math.cos(lonR), eastZ = 0;
            const northX = -Math.sin(latR)*Math.cos(lonR), northY = -Math.sin(latR)*Math.sin(lonR), northZ = Math.cos(latR);
            let srcX = acEcef.x, srcY = acEcef.y, srcZ = acEcef.z;
            if (typeof heading === 'number') {
              const headingR = heading * Math.PI/180;
              // Reciprocal of the nose heading, i.e. straight back toward the tail.
              const tailEast  = -Math.sin(headingR) * APU_TAIL_OFFSET_M;
              const tailNorth = -Math.cos(headingR) * APU_TAIL_OFFSET_M;
              srcX += eastX*tailEast + northX*tailNorth;
              srcY += eastY*tailEast + northY*tailNorth;
              srcZ += eastZ*tailEast + northZ*tailNorth;
            }
            const dx = camPos.x - srcX;
            const dy = camPos.y - srcY;
            const dz = camPos.z - srcZ;
            const dist = Math.sqrt(dx*dx + dy*dy + dz*dz);
            // Distance falloff: quieter + duller further from camera.
            const t = Math.max(0, Math.min(1, (dist - APU_DIST_NEAR) / (APU_DIST_FAR - APU_DIST_NEAR)));
            distMult = 1 - t * (1 - APU_FALLOFF_FAR_MULT);
            lpfHz = APU_LPF_NEAR_HZ + (APU_LPF_FAR_HZ - APU_LPF_NEAR_HZ) * t;
          }
                let apuFinal = baseVol * distMult;
          try {
            const acId = String(window.geofs?.aircraft?.instance?.id ?? '');
            if (APU_A330_IDS.indexOf(acId) !== -1) {
              if (window._GE_apuState === 'starting')      apuFinal *= APU_A330_STARTUP_MULT;
              else if (window._GE_apuState === 'on')       apuFinal *= APU_A330_LOOP_MULT;
              else if (window._GE_apuState === 'shutting') apuFinal *= APU_A330_SHUTDOWN_MULT;
            }
            // Exterior-only family attenuation — applies to whichever
            // clip is currently sounding (startup/loop/shutdown alike),
            // stacked on top of the A330 state-based multiplier above.
            if (view === 'exterior') {
              apuFinal *= _GE_apuExteriorAttenMult(acId);
            }
          } catch(e){}
          // Follow the shared GE-Ultimate volume slider, same as every other
          // pack's engine mix and the GPWS alerts already do.
          const apuSliderVol = (typeof window._GE_customVolume === 'number' && !isNaN(window._GE_customVolume)) ? window._GE_customVolume : 1.0;
          apuFinal *= apuSliderVol;
          gain.gain.setTargetAtTime(apuFinal, now, 0.2);
          lpf.frequency.setTargetAtTime(lpfHz, now, 0.2);
          if (interiorLpf) interiorLpf.frequency.setTargetAtTime(interiorTargetHz, now, 0.2);
        } catch(e){}
      }
      setInterval(_GE_apuSpatialTick, 100);
      // --- S / P mute + pause -------------------------------------------
      // Follows the same S (_GE_gpwsSKeyMuted) / P (geofs.isPaused()) signals
      // as AC and Cabin Safety Audio. MUTE fades the APU master gain to 0 but
      // keeps clips running silently (state/position survive); PAUSE suspends
      // the APU's AudioContext after the fade, freezing playback (incl.
      // startup->loop->shutdown hand-offs) to resume on unpause.
      const APU_MUTE_FADE_TC = 0.02;          // s — fade time constant for mute/pause
      const APU_PAUSE_SUSPEND_DELAY_MS = 150; // let the fade land before suspending
      function _GE_apuIsMuted(){
        return !!window._GE_gpwsSKeyMuted;
      }
      function _GE_apuIsPaused(){
        try {
          return !!(window.geofs && typeof window.geofs.isPaused === 'function' && window.geofs.isPaused());
        } catch(e){
          return false;
        }
      }
      function _GE_apuApplyMutePause(){
        try {
          const actx = window._GE_apuCtx;
          const master = window._GE_apuMaster;
          if (!actx || !master) return;
          const muted = _GE_apuIsMuted();
          const paused = _GE_apuIsPaused();
          // Fade the bus only when the wanted level changes (or when the
          // master node itself was rebuilt, e.g. after a failed audio load
          // was retried) instead of stacking a new ramp every tick.
          const target = (muted || paused) ? 0 : 1;
          if (window._GE_apuMasterTarget !== target || window._GE_apuMasterTargetNode !== master) {
            window._GE_apuMasterTarget = target;
            window._GE_apuMasterTargetNode = master;
            master.gain.setTargetAtTime(target, actx.currentTime, APU_MUTE_FADE_TC);
          }
          // Suspend/resume only on the paused edge — never every tick — so
          // this doesn't keep poking a context the browser's autoplay policy
          // hasn't unlocked yet.
          if (paused && !window._GE_apuPauseApplied) {
            window._GE_apuPauseApplied = true;
            clearTimeout(window._GE_apuSuspendTimer);
            window._GE_apuSuspendTimer = setTimeout(() => {
              if (window._GE_apuPauseApplied) { try { actx.suspend(); } catch(e){} }
            }, APU_PAUSE_SUSPEND_DELAY_MS);
          } else if (!paused && window._GE_apuPauseApplied) {
            window._GE_apuPauseApplied = false;
            clearTimeout(window._GE_apuSuspendTimer);
            try { actx.resume(); } catch(e){}
          }
        } catch(e){}
      }
      window._GE_apuApplyMutePause = _GE_apuApplyMutePause;
      if (!window._GE_apuMuteTickInstalled) {
        window._GE_apuMuteTickInstalled = true;
        setInterval(() => { try { window._GE_apuApplyMutePause(); } catch(e){} }, 50);
      }
      const apuRow = document.createElement('div');
      apuRow.id = 'ge90-apu-row';
      apuRow.style.display = 'flex';
      apuRow.style.alignItems = 'center';
      apuRow.style.justifyContent = 'space-between';
      apuRow.style.gap = '10px';
      apuRow.style.margin = '10px 0 0 0';
      const apuLabel = document.createElement('div');
      apuLabel.textContent = 'APU:';
      apuLabel.style.color = 'white';
      apuLabel.style.fontSize = '12px';
      apuLabel.style.fontFamily = 'Arial, sans-serif';
      const apuBtn = document.createElement('button');
      apuBtn.id = 'ge90-apu-toggle';
      apuBtn.type = 'button';
      apuBtn.style.fontSize = '12px';
      apuBtn.style.fontFamily = 'Arial, sans-serif';
      apuBtn.style.fontWeight = 'bold';
      apuBtn.style.padding = '3px 10px';
      apuBtn.style.borderRadius = '6px';
      apuBtn.style.border = '1px solid rgba(255,255,255,0.3)';
      apuBtn.style.cursor = 'pointer';
      apuBtn.style.minWidth = '92px';
      apuBtn.style.textAlign = 'center';
      const APU_STATE_STYLES = {
        off:      { text: 'OFF',          background: 'rgba(220,70,70,0.35)',  color: '#ff9d9d' },
        starting: { text: 'STARTING',     background: 'rgba(230,160,40,0.35)', color: '#ffd27f' },
        on:       { text: 'ON',           background: 'rgba(70,200,120,0.35)', color: '#8fffb0' },
        shutting: { text: 'SHUTTING OFF', background: 'rgba(230,160,40,0.35)', color: '#ffd27f' }
      };
      function refreshApuBtn(){
        const s = APU_STATE_STYLES[window._GE_apuState] || APU_STATE_STYLES.off;
        apuBtn.textContent = s.text;
        apuBtn.style.background = s.background;
        apuBtn.style.color = s.color;
      }
      function setApuState(state){
        window._GE_apuState = state;
        window._GE_apuOn = (state === 'on');
        try { localStorage.setItem('GE90_apuOn', String(window._GE_apuOn)); } catch(e){}
        refreshApuBtn();
      }
      function _GE_apuActuallyStart(){
        const buffers = window._GE_apuBuffers;
        if (!buffers) return;
        _GE_apuPlayBuffer(buffers.startup, false, () => {
          if (window._GE_apuState !== 'starting') return;
          setApuState('on');
          _GE_apuPlayBuffer(buffers.loop, true);
        });
      }
      function startApu(){
        if (window._GE_apuState !== 'off') return;
        setApuState('starting');
        if (window._GE_apuAudioReady) {
          _GE_apuActuallyStart();
        } else {
          // Buffers are still fetching/decoding — kick off the moment
          // they're ready instead of dropping the click.
          window._GE_apuOnBuffersReady = _GE_apuActuallyStart;
          loadApuAudio();
        }
      }
      function stopApu(){
        if (window._GE_apuState !== 'on') return;
        try {
          const src = window._GE_apuActiveSrc;
          if (src) { src.onended = null; src.stop(); }
        } catch(e){}
        setApuState('shutting');
        const buffers = window._GE_apuBuffers;
        if (buffers) {
          _GE_apuPlayBuffer(buffers.shutdown, false, () => {
            if (window._GE_apuState !== 'shutting') return;
            setApuState('off');
          });
        } else {
          setApuState('off');
        }
      }
      refreshApuBtn();
      apuBtn.addEventListener('click', () => {
        // A click is a user gesture, so this is the reliable place to resume
        // a suspended AudioContext. Skipped while paused (P-key wiring owns
        // suspend/resume then).
        try { if (window._GE_apuCtx && window._GE_apuCtx.state === 'suspended' && !_GE_apuIsPaused()) window._GE_apuCtx.resume(); } catch(e){}
        // Mid-transition clicks (starting/shutting) are ignored — the APU
        // can't be interrupted once it's spooling up or winding down.
        if (window._GE_apuState === 'off') {
          startApu();
        } else if (window._GE_apuState === 'on') {
          stopApu();
        }
      });
      apuRow.appendChild(apuLabel);
      apuRow.appendChild(apuBtn);
      wrap.appendChild(apuRow);
      // If the panel is (re)built while the APU was already left running
      // (e.g. state persisted from before a hide/show), just resume the
      // loop directly rather than replaying the startup clip.
      if (window._GE_apuState === 'on' && !window._GE_apuActiveSrc) {
        if (window._GE_apuAudioReady) {
          _GE_apuPlayBuffer(window._GE_apuBuffers.loop, true);
        } else {
          window._GE_apuOnBuffersReady = () => _GE_apuPlayBuffer(window._GE_apuBuffers.loop, true);
        }
      }
      const safetyDivider = document.createElement('div');
      safetyDivider.style.borderTop = '1px solid rgba(255,255,255,0.2)';
      safetyDivider.style.margin = '10px 0 8px 0';
      wrap.appendChild(safetyDivider);
      const safetyTitle = document.createElement('div');
      safetyTitle.textContent = 'Cabin Safety Audio';
      safetyTitle.style.color = 'white';
      safetyTitle.style.fontSize = '12px';
      safetyTitle.style.fontFamily = 'Arial, sans-serif';
      safetyTitle.style.fontWeight = 'bold';
      safetyTitle.style.marginBottom = '6px';
      wrap.appendChild(safetyTitle);
      const safetySearchInput = document.createElement('input');
      safetySearchInput.type = 'text';
      safetySearchInput.id = 'ge90-safety-search';
      safetySearchInput.placeholder = 'Search airline...';
      safetySearchInput.autocomplete = 'off';
      safetySearchInput.style.width = '100%';
      safetySearchInput.style.boxSizing = 'border-box';
      safetySearchInput.style.padding = '5px 8px';
      safetySearchInput.style.borderRadius = '6px';
      safetySearchInput.style.border = '1px solid rgba(255,255,255,0.3)';
      safetySearchInput.style.background = 'rgba(255,255,255,0.12)';
      safetySearchInput.style.color = 'white';
      safetySearchInput.style.fontSize = '12px';
      safetySearchInput.style.fontFamily = 'Arial, sans-serif';
      safetySearchInput.style.marginBottom = '6px';
      wrap.appendChild(safetySearchInput);
      const safetyListEl = document.createElement('div');
      safetyListEl.id = 'ge90-safety-list';
      safetyListEl.style.display = 'flex';
      safetyListEl.style.flexDirection = 'column';
      safetyListEl.style.gap = '4px';
      safetyListEl.style.maxHeight = '170px';
      safetyListEl.style.overflowY = 'auto';
      wrap.appendChild(safetyListEl);
      const GE90_safetyAudios = [
        { airline: "Vietnam Airlines", url: "https://od.lk/s/NTRfNDAyMDM3NzVf/Phim%20Hu%CC%9Bo%CC%9B%CC%81ng%20Da%CC%82%CC%83n%20An%20Toa%CC%80n%20Bay%202025_%20Chuye%CC%82%CC%81n%20Bay%20No%CC%9B%CC%89%20Hoa%20%28The%20Blossoming%29%20-%20Vietnam%20Airlines.mp3" },
        { airline: "Qantas", url: "https://od.lk/s/NTRfNDAyMDM3Nzdf/Qantas%20Safety%20Video%202024.mp3" },
        { airline: "Virgin Atlantic", url: "https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Sound-Pack-Ultimate-/main/Virgin%20Atlantic%20Safety%20Video%20(1).wav" },
        { airline: "Air India", url: "https://od.lk/s/NTRfNDAyMDM3Nzhf/Safety%20Mudras%20-%20Air%20India%27s%20Inflight%20Safety%20Video.mp3" },
        { airline: "Sri Lanka Airlines", url: "https://od.lk/s/NTRfNDAyMDM3Nzlf/SriLankan%20Airlines%20_%20Onboard%20Safety%20Video%202024.mp3" },
        { airline: "SWSS", url: "https://od.lk/s/NTRfNDAyMDM3ODBf/SWISS%20Safety%20Video%202025%20_%20SWISS.mp3" },
        { airline: "United", url: "https://od.lk/s/NTRfNDAyMDM3ODNf/United%20%E2%80%94%20Safety%20in%20Motion.mp3" },
        { airline: "THAI Airways", url: "https://od.lk/s/NTRfNDAyMDM3ODJf/Thai%20Airways%20New%20B787-9%20Safety%20Video.mp3" },
        { airline: "Malaysian Airlines", url: "https://od.lk/s/NTRfNDAyMDM3ODRf/Where%20Culture%20Takes%20Flight%20_%20Malaysia%20Airlines%20In-flight%20Safety%20Video%202025.mp3" },
        { airline: "STARLUX", url: "https://od.lk/s/NTRfNDAyMDM3ODVf/%E6%98%9F%E5%AE%87%E8%88%AA%E7%A9%BA%E6%A9%9F%E4%B8%8A%E5%AE%89%E5%85%A8%E5%BD%B1%E7%89%87%20%EF%BC%8D%20STARWONDERERS%20%E6%98%9F%E6%8E%A2%E8%80%85%20_%20Safety%20Film%EF%BD%9CSTARLUX%20Airlines.mp3" },
        { airline: "Cathay Pacific", url: "https://od.lk/s/NTRfNDAyMDM3OTRf/CATHAY%20PACIFIC%20SAFETY%20VIDEO%202024%20%E5%9C%8B%E6%B3%B0%E8%88%AA%E7%A9%BA%E9%A3%9B%E8%A1%8C%E5%AE%89%E5%85%A8%E7%A4%BA%E7%AF%84%E7%9F%AD%E7%89%87%202024.mp3" },
        { airline: "British Airways", url: "https://od.lk/s/NTRfNDAyMDM3OTNf/British%20Airways%20_%20Safety%20Video%202024%20_%20May%20We%20Haveth%20One%E2%80%99s%20Attention.mp3" },
        { airline: "ANA", url: "https://od.lk/s/NTRfNDAyMDM3OTJf/ANA%20Safety%20Video%20featuring%20Poke%CC%81mon.mp3" },
        { airline: "Air Canada", url: "https://od.lk/s/NTRfNDAyMDM3ODlm/Air%20Canada_%20New%20Safety%20Video%20_%20Nouvelle%20vide%CC%81o%20de%20se%CC%81curite%CC%81.mp3" },
        { airline: "Air China", url: "https://od.lk/s/NTRfNDAyMDM3OTBf/Air%20China%20Safety%20Video%20%E5%9B%BD%E8%88%AA%20%E5%AE%89%E5%85%A8%E9%A1%BB%E7%9F%A5%20%E4%B8%AD%E5%9B%BD%E5%9B%BD%E9%9A%9B%E8%88%AA%E7%A9%BA%20%E6%A9%9F%E5%86%85%E5%AE%89%E5%85%A8%E3%83%92%E3%82%99%E3%83%86%E3%82%99%E3%82%AA%202023.mp3" },
        { airline: "American Airlines", url: "https://od.lk/s/NTRfNDAyMDM3OTFf/American%20Airlines%20Safety%20Video.mp3" },
        { airline: "Qatar Airways", url: "https://od.lk/s/NTRfNDAyMDM3ODhf/A%20safety%20video%20coming%20from%20the%20Hart%20_%20Qatar%20Airways.mp3" },
        { airline: "EVA Air", url: "https://od.lk/s/NTRfNDAyMDM3ODZf/2026%20EVA%20Air%20Safety%20Video%20%E3%80%8AFlying%20with%20EVA%20AIR%E3%80%8B.mp3" },
        { airline: "Delta", url: "https://od.lk/s/NTRfNDAyMDM3ODdf/A%20Hundred%20Years%20of%20Safety%20-%20Delta%27s%202025%20Centennial%20Safety%20Video.mp3" },
        { airline: "LATAM", url: "https://od.lk/s/NTRfNDAyMDM3OTdf/Experience%20South%20America%20with%20our%20safety%20video%21.mp3" },
        { airline: "Hainan Airlines", url: "https://od.lk/s/NTRfNDAyMDM4MDBf/Hainan%20Airlines%20Safety%20Video%20%282024%29.mp3" },
        { airline: "Air New Zealand", url: "https://od.lk/s/NTRfNDAyMDM4MDFf/It%E2%80%99s%20Game%20on%20for%20Safety%20%23AirNZSafetyVideo.mp3" },
        { airline: "ITA Airways", url: "https://od.lk/s/NTRfNDAyMDM4MDNf/ITA%20Airways%20_%20Istruzioni%20di%20sicurezza%20a%20bordo%20_%20Safety%20video.mp3" },
        { airline: "Japan Airlines", url: "https://od.lk/s/NTRfNDAyMDM4MDRf/JAL%20%28Japan%20Airlines%29%20in-flight%20safety%20video%2C%20New%20Version.mp3" },
        { airline: "Jetstar", url: "https://od.lk/s/NTRfNDAyMDM4MDVf/Jetstar%20A320%20Safety%20demo.mp3" },
        { airline: "KLM", url: "https://od.lk/s/NTRfNDAyMDM4MDZf/KLM%20Flight%20Safety.mp3" },
        { airline: "Korean Air", url: "https://od.lk/s/NTRfNDAyMDM4MDdf/Korean%20Air%2C%20in-flight%20safety%20video.mp3" },
        { airline: "LOT Polish Airlines", url: "https://od.lk/s/NTRfNDAyMDM4MDhf/New%20LOT%20Polish%20Airlines%20Safety%20Video.mp3" },
        { airline: "Etihad", url: "https://od.lk/s/NTRfNDAyMDM4MTBf/New%20Safety%20Video%20_%20Etihad.mp3" },
        { airline: "Turkish Airlines", url: "https://od.lk/s/NTRfNDAyMDM4MTFf/NEW%20Turkish%20Airlines%20LEGO%20Safety%20Video.mp3" },
        { airline: "Emirates", url: "https://od.lk/s/NTRfNDAyMDM4MTJf/Our%20New%20No-Nonsense%20Safety%20Video%20_%20Emirates.mp3" },
        { airline: "Lufthansa", url: "https://od.lk/s/NTRfNDAyMDM4MTRf/Our%20new%20Safety%20Video%20_%20Lufthansa.mp3" },
        { airline: "Philippines Airlines", url: "https://od.lk/s/NTRfNDAyMDM4MTZf/Philippine%20Airlines%20Inflight%20Safety%20Video%20%23PALSafetynovela%20_%20Care%20That%20Comes%20From%20The%20Heart.mp3" },
        { airline: "Singapore Airlines", url: "https://od.lk/s/NTRfNDAyMDM4MThf/Welcome%20on%20board%C2%A0_%20Introducing%20Singapore%20Airlines%E2%80%99%20New%20In-flight%20Safety%20Video_.mp3" }
      ];
      const safetyAudioPlayer = new Audio();
      safetyAudioPlayer._GE90_TAG = true;
      safetyAudioPlayer.dataset.geofsAllow = 'true';
      safetyAudioPlayer._GE_safetyViewGated = true;
      safetyAudioPlayer.style.display = 'none';
      document.body.appendChild(safetyAudioPlayer);
      let GE90_currentSafetyUrl = '';
      function GE90_getSafetyCameraView() {
        try {
          const cam = window.geofs?.camera;
          if (!cam) return 'exterior';
          let s = cam.currentModeName || cam.currentView || cam.currentDefinition?.name || '';
          s = String(s).toLowerCase();
          if (!s) return 'exterior';
          if (
            s.includes('cockpit') ||
            s.includes('jumpseat') ||
            s.includes('jump seat') ||
            s.includes('virtual cockpit') ||
            s.includes('vc')
          ) return 'cockpit';
          if (
            (s.includes('left wing') || s.includes('right wing') || s.includes('right') || s.includes('left')) &&
            (s.includes('cfm') || s.includes('iaev') || s.includes('v2500') || s.includes('engine'))
          ) return 'wingEngine';
          if (s.includes('wingr1') || s.includes('wingl1') || s.includes('business class')) return 'wingEngine';
          if (
            s.includes('wing 2') ||
            s.includes('wing2') ||
            s.includes('left wing') ||
            s.includes('right wing') ||
            s.includes('wingr2') ||
            s.includes('wingl2')
          ) return 'wing2';
          return 'exterior';
        } catch (e) {
          return 'exterior';
        }
      }
      function GE90_isSafetyAudibleView() {
        const v = GE90_getSafetyCameraView();
        return v === 'wing2' || v === 'wingEngine';
      }
      let GE90_safetyBaseVolume = 0.35;
      function GE90_setSafetyBaseVolume(v) {
        GE90_safetyBaseVolume = v;
        GE90_applySafetyVolume();
      }
     function GE90_applySafetyVolume() {
  try {
    const globallyMuted = !!window._GE_gpwsSKeyMuted;
    const globallyPaused = !!(window.geofs && typeof window.geofs.isPaused === 'function' && window.geofs.isPaused());
    if (globallyMuted || globallyPaused) {
      if (!safetyAudioPlayer.paused) { try { safetyAudioPlayer.pause(); } catch(e){} }
      safetyAudioPlayer.volume = 0;
      return;
    }
    if (GE90_currentSafetyUrl && safetyAudioPlayer.paused && safetyAudioPlayer.src) {
      const p = safetyAudioPlayer.play();
      if (p && typeof p.catch === 'function') p.catch(()=>{});
    }
    safetyAudioPlayer.volume = GE90_isSafetyAudibleView() ? GE90_safetyBaseVolume : 0;
  } catch(e){}
}
      let GE90_safetyViewPollTimer = null;
      function GE90_startSafetyViewPolling() {
        if (GE90_safetyViewPollTimer) return;
        GE90_safetyViewPollTimer = setInterval(GE90_applySafetyVolume, 200);
      }
      function GE90_stopSafetyViewPolling() {
        if (!GE90_safetyViewPollTimer) return;
        clearInterval(GE90_safetyViewPollTimer);
        GE90_safetyViewPollTimer = null;
      }
      function GE90_renderSafetyAudios(filter) {
        safetyListEl.innerHTML = '';
        const q = String(filter || '').toLowerCase();
        const filtered = GE90_safetyAudios.filter(v => v.airline.toLowerCase().includes(q));
        if (filtered.length === 0) {
          const empty = document.createElement('div');
          empty.textContent = 'No matching airline';
          empty.style.color = 'rgba(255,255,255,0.6)';
          empty.style.fontSize = '12px';
          empty.style.fontFamily = 'Arial, sans-serif';
          empty.style.padding = '4px 2px';
          safetyListEl.appendChild(empty);
          return;
        }
        filtered.forEach(v => {
          const isPlaying = (v.url === GE90_currentSafetyUrl && !safetyAudioPlayer.paused);
          const item = document.createElement('div');
          item.className = 'ge90-safety-item';
          item.setAttribute('data-url', v.url);
          item.style.display = 'flex';
          item.style.alignItems = 'center';
          item.style.justifyContent = 'space-between';
          item.style.gap = '8px';
          item.style.padding = '5px 10px';
          item.style.borderRadius = '6px';
          item.style.border = '1px solid rgba(255,255,255,0.15)';
          item.style.cursor = 'pointer';
          item.style.background = isPlaying ? 'rgba(70,200,120,0.35)' : 'rgba(255,255,255,0.08)';
          item.addEventListener('mouseenter', () => {
            if (!(v.url === GE90_currentSafetyUrl && !safetyAudioPlayer.paused)) {
              item.style.background = 'rgba(255,255,255,0.18)';
            }
          });
          item.addEventListener('mouseleave', () => {
            item.style.background = (v.url === GE90_currentSafetyUrl && !safetyAudioPlayer.paused)
              ? 'rgba(70,200,120,0.35)' : 'rgba(255,255,255,0.08)';
          });
          const nameSpan = document.createElement('div');
          nameSpan.textContent = v.airline;
          nameSpan.style.color = 'white';
          nameSpan.style.fontSize = '12px';
          nameSpan.style.fontFamily = 'Arial, sans-serif';
          const dot = document.createElement('div');
          dot.style.width = '7px';
          dot.style.height = '7px';
          dot.style.borderRadius = '50%';
          dot.style.background = '#8fffb0';
          dot.style.flexShrink = '0';
          dot.style.display = isPlaying ? 'block' : 'none';
          item.appendChild(nameSpan);
          item.appendChild(dot);
          item.addEventListener('click', () => GE90_playSafetySequence(v.url));
          safetyListEl.appendChild(item);
        });
      }
      function GE90_updateSafetyUI() {
        GE90_renderSafetyAudios(safetySearchInput.value);
      }
      const GE90_playSafetySequence = async (targetUrl) => {
        if (GE90_currentSafetyUrl === targetUrl && !safetyAudioPlayer.paused) {
          safetyAudioPlayer.pause();
          GE90_stopSafetyViewPolling();
          GE90_updateSafetyUI();
          return;
        }
        GE90_currentSafetyUrl = targetUrl;
        GE90_startSafetyViewPolling();
        GE90_updateSafetyUI();
        try {
          GE90_setSafetyBaseVolume(1);
          safetyAudioPlayer.src = targetUrl;
          await safetyAudioPlayer.play();
          safetyAudioPlayer.onended = () => {
            GE90_currentSafetyUrl = '';
            GE90_stopSafetyViewPolling();
            GE90_updateSafetyUI();
          };
        } catch (err) {
          console.error('[GE90 Ultimate] Cabin Safety Audio playback error:', err);
        }
      };
      safetySearchInput.addEventListener('input', () => {
        GE90_renderSafetyAudios(safetySearchInput.value);
      });
      GE90_renderSafetyAudios('');
const liveAtcDB = {
    kjfk: {
        freqs: {
            "Clearance Delivery": [
                "https://www.liveatc.net/hlisten.php?mount=kjfk_del&icao=kjfk",
                "https://www.liveatc.net/hlisten.php?mount=kjfk_del2&icao=kjfk",
                "https://www.liveatc.net/hlisten.php?mount=kjfk_del3&icao=kjfk"
            ],
            "Ground": [
                "https://www.liveatc.net/hlisten.php?mount=kjfk_gnd&icao=kjfk",
                "https://www.liveatc.net/hlisten.php?mount=kjfk_gnd2&icao=kjfk"
            ],
            "Tower": [
                "https://www.liveatc.net/hlisten.php?mount=kjfk_twr&icao=kjfk",
                "https://www.liveatc.net/hlisten.php?mount=kjfk_twr2&icao=kjfk"
            ]
        }
    },
    klax: {
        freqs: {
            "ATIS": [
                "https://www.liveatc.net/hlisten.php?mount=klax4n_atis_dep&icao=klax"
            ],
            "Clearance Delivery": [
                "https://www.liveatc.net/hlisten.php?mount=klax3n_del&icao=klax"
            ],
            "Ground": [
                "https://www.liveatc.net/hlisten.php?mount=klax1&icao=klax",
                "https://www.liveatc.net/hlisten.php?mount=klax_gnd&icao=klax",
                "https://www.liveatc.net/hlisten.php?mount=klax5&icao=klax"
            ],
            "Tower": [
                "https://www.liveatc.net/hlisten.php?mount=klax3&icao=klax",
                "https://www.liveatc.net/hlisten.php?mount=khhr1_klax_twr_n&icao=klax",
                "https://www.liveatc.net/hlisten.php?mount=klax_twr&icao=klax",
                "https://www.liveatc.net/hlisten.php?mount=klax4&icao=klax",
                "https://www.liveatc.net/hlisten.php?mount=khhr1_klax_twr_s&icao=klax"
             ],
             "Departure": [
                "https://www.liveatc.net/hlisten.php?mount=klax7&icao=klax",
                "https://www.liveatc.net/hlisten.php?mount=klax3n_dep_125200&icao=klax"
            ],
            "Approach": [
                "https://www.liveatc.net/hlisten.php?mount=klax3n_nw_feeder&icao=klax",
                "https://www.liveatc.net/hlisten.php?mount=khhr1_app_124500&icao=klax",
                "https://www.liveatc.net/hlisten.php?mount=klax6&icao=klax",
                "https://www.liveatc.net/hlisten.php?mount=klax3n_app_fin_s&icao=klax"
            ]
        }
    },
    ksfo: {
        freqs: {
            "ATIS": [
                "https://www.liveatc.net/hlisten.php?mount=ksfo_atis&icao=ksfo"
            ],
            "Clearance Delivery/Ground/Tower": [
                "https://www.liveatc.net/hlisten.php?mount=ksfo_gnd2&icao=ksfo"
            ],
            "Ground": [
                "https://www.liveatc.net/hlisten.php?mount=ksfo_gnd&icao=ksfo",
            ],
            "Tower": [
                "https://www.liveatc.net/hlisten.php?mount=ksfo_twr&icao=ksfo",
             ],
             "Departure": [
                "https://www.liveatc.net/hlisten.php?mount=ksfo_dep1&icao=ksfo",
                "https://www.liveatc.net/hlisten.php?mount=ksfo_dep2&icao=ksfo"
            ],
            "Approach": [
                "https://www.liveatc.net/hlisten.php?mount=ksfo_app2_l&icao=ksfo",
                "https://www.liveatc.net/hlisten.php?mount=ksfo_app2&icao=ksfo",
                "https://www.liveatc.net/hlisten.php?mount=ksfo_app2_r&icao=ksfo",
            ],
        }
    },
    wsss: {
        freqs: {
            "Ground/Tower/Approach/Radar": [
                "https://www.liveatc.net/hlisten.php?mount=wsss2&icao=wsss"
            ],
            "Delivery/Ground/Approach/Radar": [
                "https://www.liveatc.net/hlisten.php?mount=wsss3&icao=wsss"
            ]
        }
    },
    rjtt: {
        freqs: {
            "Clearance Delivery/Ground": [
                "https://www.liveatc.net/hlisten.php?mount=rjtt_gnd&icao=rjtt"
            ],
            "Tower": [
                "https://www.liveatc.net/hlisten.php?mount=rjtt_twr&icao=rjtt"
            ],
            "Approach": [
                "https://www.liveatc.net/hlisten.php?mount=rjtt_app&icao=rjtt"
            ],
            "Departure": [
                "https://www.liveatc.net/hlisten.php?mount=rjtt_dep&icao=rjtt"
            ]
        }
    },
    vhhh: {
        freqs: {
            "Tower": [
                "https://www.liveatc.net/hlisten.php?mount=vhhh_twr&icao=vhhh"
            ],
            "Ground": [
                "https://www.liveatc.net/hlisten.php?mount=vhhh_gnd&icao=vhhh"
            ],
            "Approach": [
                "https://www.liveatc.net/hlisten.php?mount=vhhh_app_1191&icao=vhhh"
            ],
            "Approach/Departure/Director/Zone": [
                "https://www.liveatc.net/hlisten.php?mount=vhhh5&icao=vhhh"
            ]
        }
    },
    rjaa: {
        freqs: {
            "Clearance Delivery": [
                "https://www.liveatc.net/hlisten.php?mount=rjaa_del1&icao=rjaa"
            ],
            "Ground": [
                "https://www.liveatc.net/hlisten.php?mount=rjaa_gnd1&icao=rjaa"
            ],
            "Tower": [
                "https://www.liveatc.net/hlisten.php?mount=rjaa_twr&icao=rjaa"
            ],
            "Approach": [
                "https://www.liveatc.net/hlisten.php?mount=rjaa_app_s&icao=rjaa"
            ],
            "Departure": [
                "https://www.liveatc.net/hlisten.php?mount=rjaa_dep1&icao=rjaa"
            ]
        }
    },
    eham: {
        freqs: {
            "Tower": [
                "https://www.liveatc.net/hlisten.php?mount=eham01_twr_main2&icao=eham"
            ],
            "Approach/Departure": [
                "https://www.liveatc.net/hlisten.php?mount=eham4&icao=eham"
            ],
            "Startup/Outbound Planner": [
                "https://www.liveatc.net/hlisten.php?mount=eham01_startup&icao=eham"
            ]
        }
    },
    katl: {
        freqs: {
            "Tower": [
                "https://www.liveatc.net/hlisten.php?mount=katl_twr&icao=katl"
            ],
            "Ground": [
                "https://www.liveatc.net/hlisten.php?mount=katl_gnd&icao=katl"
            ],
            "Approach": [
                "https://www.liveatc.net/hlisten.php?mount=kcco1_atl_app&icao=katl"
            ]
        }
    },
    kord: {
        freqs: {
            "Ground": [
                "https://www.liveatc.net/hlisten.php?icao=kord&mount=kord1"
            ],
            "Tower": [
                "https://www.liveatc.net/hlisten.php?mount=kord7&icao=kord"
            ],
            "Approach": [
                "https://www.liveatc.net/hlisten.php?mount=kord_app&icao=kord"
            ]
        }
    },
    kdfw: {
        freqs: {
            "Tower (East)": [
                "https://www.liveatc.net/hlisten.php?mount=kdfw1_twr1_e&icao=kdfw"
            ],
            "Tower (West)": [
                "https://www.liveatc.net/hlisten.php?mount=kdfw1_twr1_w&icao=kdfw"
            ],
            "Ground (East)": [
                "https://www.liveatc.net/hlisten.php?mount=kdfw1_gnd_e_12165&icao=kdfw"
            ],
            "Ground (West)": [
                "https://www.liveatc.net/hlisten.php?mount=kdfw1_gnd_w&icao=kdfw"
            ]
        }
    },
    kden: {
        freqs: {
            "Clearance Delivery": [
                "https://www.liveatc.net/hlisten.php?mount=kden1_9&icao=kden"
            ],
            "Tower": [
                "https://www.liveatc.net/hlisten.php?mount=kden1_3&icao=kden"
            ],
            "Approach": [
                "https://www.liveatc.net/hlisten.php?icao=kden&mount=kden1_1"
            ],
            "Departure": [
                "https://www.liveatc.net/hlisten.php?mount=kden1_2&icao=kden"
            ]
        }
    },
    cyyz: {
        freqs: {
            "Clearance Delivery": [
                "https://www.liveatc.net/hlisten.php?mount=cyyz4&icao=cyyz"
            ],
            "Ground": [
                "https://www.liveatc.net/hlisten.php?mount=cyyz5&icao=cyyz"
            ],
            "Tower": [
                "https://www.liveatc.net/hlisten.php?icao=cyyz&mount=cyyz7"
            ],
            "Arrival": [
                "https://www.liveatc.net/hlisten.php?mount=cyyz6&icao=cyyz"
            ],
            "Departure": [
                "https://www.liveatc.net/hlisten.php?icao=cyyz&mount=cyyz8"
            ]
        }
    },
    cyvr: {
        freqs: {
            "Tower": [
                "https://www.liveatc.net/hlisten.php?mount=cyvr1_twr&icao=cyvr"
            ],
            "Approach/Departure": [
                "https://www.liveatc.net/hlisten.php?mount=cyvr1_app&icao=cyvr"
            ],
            "Delivery/Ground/Tower/Approach": [
                "https://www.liveatc.net/hlisten.php?mount=cyvr_s&icao=cyvr"
            ]
        }
    },
    yssy: {
        freqs: {
            "Tower": [
                "https://www.liveatc.net/hlisten.php?mount=yssy_twr&icao=yssy"
            ],
            "Approach (North/East)": [
                "https://www.liveatc.net/hlisten.php?mount=yssy1_app_n&icao=yssy"
            ],
            "Departure (North/East)": [
                "https://www.liveatc.net/hlisten.php?mount=yssy1_dep_ne&icao=yssy"
            ],
            "Departure (South/West)": [
                "https://www.liveatc.net/hlisten.php?mount=yssy1_dep_s&icao=yssy"
            ]
        }
    },
    engm: {
        freqs: {
            "Tower": [
                "https://www.liveatc.net/hlisten.php?mount=engm4&icao=engm"
            ],
            "Approach/Polaris ACC": [
                "https://www.liveatc.net/hlisten.php?mount=eneg2&icao=engm"
            ]
        }
    },
    essa: {
        freqs: {
            "Approach/Departure": [
                "https://www.liveatc.net/hlisten.php?icao=essa&mount=essa_app"
            ],
            "Sweden Control": [
                "https://www.liveatc.net/hlisten.php?icao=essa&mount=essa_sweden_control"
            ]
        }
    },
    eidw: {
        freqs: {
            "Clearance Delivery": [
                "https://www.liveatc.net/hlisten.php?mount=eidw_del"
            ],
            "Ground": [
                "https://www.liveatc.net/hlisten.php?mount=eidw_gnd_121800"
            ],
            "Tower": [
                "https://www.liveatc.net/hlisten.php?mount=eidw_twr_118600&icao=eidw"
            ],
            "Approach": [
                "https://www.liveatc.net/hlisten.php?mount=eidw_app_121100&icao=eidw"
            ],
            "Combined (Del/Gnd/Twr/App/Centre)": [
                "https://www.liveatc.net/hlisten.php?mount=eidw8&icao=eidw",
                "https://www.liveatc.net/hlisten.php?mount=eidw3"
            ]
        }
    },
    ebbr: {
        freqs: {
            "Arrival": [
                "https://www.liveatc.net/hlisten.php?icao=ebbr&mount=ebbr_arr"
            ],
            "Tower (East/Rwy 25L)": [
                "https://www.liveatc.net/hlisten.php?icao=ebbr&mount=ebbr_twr_e"
            ],
            "Brussels Control": [
                "https://www.liveatc.net/hlisten.php?mount=ebbr_ebbu"
            ]
        }
    },
    lppt: {
        freqs: {
            "Approach": [
                "https://www.liveatc.net/hlisten.php?mount=lppt_app"
            ]
        }
    },
    lgav: {
        freqs: {
            "Approach/Radar": [
                "https://www.liveatc.net/hlisten.php?icao=lgav&mount=lgav2"
            ]
        }
    },
    // Added 12 popular airports (US majors + requested international hubs).
    // EGLL/LFPG/EDDF are excluded (illegal to broadcast ATC audio there);
    // OMDB has no active volunteer feed — so solid US hubs were used instead.
    kmia: { // Miami International
        freqs: {
            "Ground": [
                "https://www.liveatc.net/hlisten.php?mount=kmia3_gnd&icao=kmia"
            ],
            "Tower": [
                "https://www.liveatc.net/hlisten.php?icao=KMIA&mount=kmia_twr",
                "https://www.liveatc.net/hlisten.php?mount=kmia3_twr&icao=kmia"
            ],
            "Approach": [
                "https://www.liveatc.net/hlisten.php?mount=kmia_app&icao=kmia",
                "https://www.liveatc.net/hlisten.php?icao=kmia&mount=kmia_app2"
            ],
            "Departure": [
                "https://www.liveatc.net/hlisten.php?mount=kmia_dep&icao=kmia"
            ]
        }
    },
    ksea: { // Seattle-Tacoma International
        freqs: {
            "Ground": [
                "https://www.liveatc.net/hlisten.php?mount=ksea_gnd"
            ],
            "Tower": [
                "https://www.liveatc.net/hlisten.php?mount=ksea_twr&icao=ksea"
            ],
            "Approach": [
                "https://www.liveatc.net/hlisten.php?mount=ksea_app"
            ]
        }
    },
    kmco: { // Orlando International
        freqs: {
            "Tower (East)": [
                "https://www.liveatc.net/hlisten.php?mount=korl_kmco_twr_118450&icao=kmco"
            ],
            "Tower (West)": [
                "https://www.liveatc.net/hlisten.php?icao=kmco&mount=kmco2"
            ],
            "Approach (Final)": [
                "https://www.liveatc.net/hlisten.php?mount=kmco6"
            ],
            "Approach (Disney Sector)": [
                "https://www.liveatc.net/hlisten.php?mount=kmco_app_disney&icao=kmco"
            ]
        }
    },
    klas: { // Las Vegas Harry Reid International
        freqs: {
            "Ground": [
                "https://www.liveatc.net/hlisten.php?icao=klas&mount=klas_gnd1",
                "https://www.liveatc.net/hlisten.php?icao=klas&mount=klas_gnd2"
            ],
            "Tower": [
                "https://www.liveatc.net/hlisten.php?mount=klas_twr1",
                "https://www.liveatc.net/hlisten.php?icao=klas&mount=klas3_twr",
                "https://www.liveatc.net/hlisten.php?mount=klas4_twr&icao=klas"
            ],
            "Approach": [
                "https://www.liveatc.net/hlisten.php?mount=klas4_app_vfr",
                "https://www.liveatc.net/hlisten.php?mount=klas5_app_final",
                "https://www.liveatc.net/hlisten.php?mount=klas4_app_se"
            ]
        }
    },
    kbos: { // Boston Logan International
        freqs: {
            "Clearance/Ground": [
                "https://www.liveatc.net/hlisten.php?mount=kbos_gnd&icao=kbos"
            ],
            "Tower": [
                "https://www.liveatc.net/hlisten.php?mount=kbos_twr&icao=kbos"
            ],
            "Approach": [
                "https://www.liveatc.net/hlisten.php?mount=kbos_app_north&icao=kbos",
                "https://www.liveatc.net/hlisten.php?mount=kbos_app_south",
                "https://www.liveatc.net/hlisten.php?mount=kbos_final&icao=kbos"
            ]
        }
    },
    kewr: { // Newark Liberty International
        freqs: {
            "Ground": [
                "https://www.liveatc.net/hlisten.php?mount=kewr_gnd_pri&icao=kewr"
            ],
            "Tower": [
                "https://www.liveatc.net/hlisten.php?mount=kewr_twr&icao=kewr",
                "https://www.liveatc.net/hlisten.php?icao=kewr&mount=kewr2_twr"
            ],
            "Approach": [
                "https://www.liveatc.net/hlisten.php?mount=kewr_app_n&icao=kewr",
                "https://www.liveatc.net/hlisten.php?icao=kewr&mount=kewr2_app_n_arr",
                "https://www.liveatc.net/hlisten.php?mount=kewr_app_rbv"
            ]
        }
    },
    kphx: { // Phoenix Sky Harbor International
        freqs: {
            "Ground": [
                "https://www.liveatc.net/hlisten.php?mount=kphx_gnd_n1&icao=kphx",
                "https://www.liveatc.net/hlisten.php?mount=kphx_gnd_s&icao=kphx"
            ],
            "Tower": [
                "https://www.liveatc.net/hlisten.php?mount=kphx_twr_n",
                "https://www.liveatc.net/hlisten.php?mount=kphx_twr_s"
            ],
            "Approach": [
                "https://www.liveatc.net/hlisten.php?mount=kphx_app_nf&icao=kphx",
                "https://www.liveatc.net/hlisten.php?mount=kphx_app_s&icao=kphx"
            ],
            "Departure": [
                "https://www.liveatc.net/hlisten.php?mount=kphx_dep"
            ]
        }
    },
    kiad: { // Washington Dulles International
        freqs: {
            "Clearance/Ground": [
                "https://www.liveatc.net/hlisten.php?mount=kiad1_1&icao=kiad"
            ],
            "Tower/Approach/Departure": [
                "https://www.liveatc.net/hlisten.php?icao=kiad&mount=kiad3"
            ],
            "Approach (North Arrival)": [
                "https://www.liveatc.net/hlisten.php?mount=kiad2_1"
            ],
            "Approach (West Arrival)": [
                "https://www.liveatc.net/hlisten.php?mount=kiad1_5&icao=kiad"
            ],
            "Approach (Final East)": [
                "https://www.liveatc.net/hlisten.php?mount=kiad2_3&icao=kiad"
            ]
        }
    },
    kiah: { // Houston George Bush Intercontinental
        freqs: {
            "Clearance Delivery": [
                "https://www.liveatc.net/hlisten.php?mount=kiah3_del&icao=kiah"
            ],
            "Ground": [
                "https://www.liveatc.net/hlisten.php?mount=kiah2_gnd_w&icao=kiah"
            ],
            "Tower": [
                "https://www.liveatc.net/hlisten.php?mount=kiah1_1"
            ],
            "Approach": [
                "https://www.liveatc.net/hlisten.php?icao=kiah&mount=kiah1_2"
            ]
        }
    },
    kclt: { // Charlotte Douglas International
        freqs: {
            "Clearance/Ground": [
                "https://www.liveatc.net/hlisten.php?mount=kclt_gnd&icao=kclt"
            ],
            "Tower": [
                "https://www.liveatc.net/hlisten.php?icao=kclt&mount=kclt_twr"
            ],
            "Approach/Departure": [
                "https://www.liveatc.net/hlisten.php?mount=kclt2&icao=kclt"
            ],
            "Approach (Arrival)": [
                "https://www.liveatc.net/hlisten.php?mount=kclt4_arr&icao=kclt"
            ]
        }
    },
    kmsp: { // Minneapolis-St Paul International
        freqs: {
            "Ground (West)": [
                "https://www.liveatc.net/hlisten.php?mount=kmsp3_gnd_w&icao=kmsp"
            ],
            "Tower": [
                "https://www.liveatc.net/hlisten.php?mount=kmsp3_twr_12r&icao=kmsp",
                "https://www.liveatc.net/hlisten.php?mount=kmsp3_twr_12l&icao=kmsp",
                "https://www.liveatc.net/hlisten.php?mount=kmsp3_twr_1735&icao=kmsp"
            ],
            "Approach": [
                "https://www.liveatc.net/hlisten.php?mount=kmsp3_app_ne&icao=kmsp",
                "https://www.liveatc.net/hlisten.php?mount=kmsp3_app_121200&icao=kmsp"
            ]
        }
    },
    kdtw: { // Detroit Metro Wayne County
        freqs: {
            "Ground": [
                "https://www.liveatc.net/hlisten.php?icao=kdtw&mount=kdtw_gnd"
            ],
            "Tower": [
                "https://www.liveatc.net/hlisten.php?mount=kdtw_twr"
            ],
            "Approach": [
                "https://www.liveatc.net/hlisten.php?mount=kdtw_app"
            ],
            "Departure": [
                "https://www.liveatc.net/hlisten.php?mount=kdtw_dep&icao=kdtw"
            ]
        }
    }
};
const atcContainer = document.createElement('div');
atcContainer.style.display = 'flex';
atcContainer.style.flexDirection = 'column';
atcContainer.style.gap = '6px';
atcContainer.style.marginTop = '10px';
const atcTitle = document.createElement('div');
atcTitle.textContent = 'Live ATC';
atcTitle.style.color = 'white';
atcTitle.style.fontSize = '12px';
atcTitle.style.fontFamily = 'Arial, sans-serif';
atcTitle.style.fontWeight = 'bold';
atcContainer.appendChild(atcTitle);
const icaoInput = document.createElement('input');
icaoInput.placeholder = 'Airport ICAO (e.g. KJFK)';
icaoInput.title = "Type in the ICAO of your desired airport here";
icaoInput.style.width = '160px';
icaoInput.style.padding = '3px 6px';
icaoInput.style.borderRadius = '6px';
icaoInput.style.border = '1px solid rgba(255,255,255,0.3)';
icaoInput.style.background = 'rgba(255,255,255,0.12)';
icaoInput.style.color = 'white';
icaoInput.style.fontSize = '12px';
icaoInput.style.fontFamily = 'Arial, sans-serif';
atcContainer.appendChild(icaoInput);
const freqSelect = document.createElement('select');
freqSelect.style.width = '160px';
freqSelect.style.padding = '3px 6px';
freqSelect.style.borderRadius = '6px';
freqSelect.style.border = '1px solid rgba(255,255,255,0.3)';
freqSelect.style.background = 'rgba(255,255,255,0.12)';
freqSelect.style.color = 'white';
freqSelect.style.fontSize = '12px';
freqSelect.style.fontFamily = 'Arial, sans-serif';
atcContainer.appendChild(freqSelect);
icaoInput.addEventListener('input', () => {
    const icao = icaoInput.value.trim().toLowerCase();
    freqSelect.innerHTML = "";
    if (liveAtcDB[icao]) {
        Object.keys(liveAtcDB[icao].freqs).forEach(freq => {
            const opt = document.createElement('option');
            opt.value = freq;
            opt.textContent = freq;
            freqSelect.appendChild(opt);
        });
    } else {
        const opt = document.createElement('option');
        opt.value = "none";
        opt.textContent = "ATC not available";
        freqSelect.appendChild(opt);
    }
});
if (typeof window.liveATCPopup === 'undefined') window.liveATCPopup = null;
function notifyATCCloseBlocked(){
    const el = document.getElementById('ge90-atc-status');
    if (!el) return;
    el.textContent = 'Feed is still open — the browser blocked auto-close. Click its window/tab and close it manually.';
    el.style.display = 'block';
    clearTimeout(el._hideTimer);
    el._hideTimer = setTimeout(() => { el.style.display = 'none'; }, 8000);
}
function closeATCPopup(){
    const trackedRef = window.liveATCPopup;
    try {
        if (trackedRef && !trackedRef.closed) {
            trackedRef.close();
        }
    } catch(e){}
    let reacquired = null;
    try {
        const handle = window.open('', 'liveatc');
        reacquired = handle;
        if (handle && !handle.closed) handle.close();
    } catch(e){}
    window.liveATCPopup = null;
    setTimeout(() => {
        let stillOpen = false;
        try { stillOpen = !!(trackedRef && !trackedRef.closed); } catch(e){ stillOpen = true; }
        if (!stillOpen) {
            try { stillOpen = !!(reacquired && !reacquired.closed); } catch(e){ stillOpen = true; }
        }
        if (stillOpen) {
            try { trackedRef.focus(); } catch(e){}
            notifyATCCloseBlocked();
        }
    }, 300);
}
function openATCPopup(url) {
    const statusEl = document.getElementById('ge90-atc-status');
    if (statusEl) { clearTimeout(statusEl._hideTimer); statusEl.style.display = 'none'; }
    closeATCPopup();
    window.liveATCPopup = window.open(
        url,
        "liveatc",
        "width=420,height=360,menubar=no,toolbar=no"
    );
    if (!window.liveATCPopup || window.liveATCPopup.closed) {
        alert("Popup blocked! Please allow popups for GeoFS.");
    }
}
const loadBtn = document.createElement('button');
loadBtn.textContent = 'Open ATC';
loadBtn.style.fontSize = '12px';
loadBtn.style.fontFamily = 'Arial, sans-serif';
loadBtn.style.padding = '3px 10px';
loadBtn.style.borderRadius = '6px';
loadBtn.style.border = '1px solid rgba(255,255,255,0.3)';
loadBtn.style.background = 'rgba(77,163,255,0.35)';
loadBtn.style.color = '#bfe3ff';
loadBtn.style.cursor = 'pointer';
loadBtn.addEventListener('click', () => {
    const icao = icaoInput.value.trim().toLowerCase();
    const freq = freqSelect.value;
    if (!liveAtcDB[icao]) {
        alert("ATC not available");
        return;
    }
    const feeds = liveAtcDB[icao].freqs[freq];
    if (feeds.length > 1) {
        let msg = "Multiple feeds available:\n\n";
        feeds.forEach((f, i) => {
            msg += `${i+1}) ${freq} Frequency ${i+1}\n`;
        });
        msg += "\nEnter the number you want:";
        const choice = prompt(msg);
        // CANCEL or EMPTY → do nothing
        if (choice === null || choice.trim() === "") return;
        const index = Number(choice) - 1;
        // INVALID NUMBER → do nothing
        if (isNaN(index) || index < 0 || index >= feeds.length) return;
        openATCPopup(feeds[index]);
        return;
    }
    // Single feed → open directly
    openATCPopup(feeds[0]);
});
atcContainer.appendChild(loadBtn);
// ---------------------------------------------------------------
// Close popup button
// ---------------------------------------------------------------
const closeBtn = document.createElement('button');
closeBtn.textContent = 'Close ATC';
closeBtn.style.fontSize = '12px';
closeBtn.style.fontFamily = 'Arial, sans-serif';
closeBtn.style.padding = '3px 10px';
closeBtn.style.borderRadius = '6px';
closeBtn.style.border = '1px solid rgba(255,255,255,0.3)';
closeBtn.style.background = 'rgba(255,255,255,0.25)';
closeBtn.style.color = '#ffffff';
closeBtn.style.cursor = 'pointer';
closeBtn.addEventListener('click', () => {
    closeATCPopup();
});
atcContainer.appendChild(closeBtn);
// Hidden by default; shown briefly by notifyATCCloseBlocked() only when
// the browser has blocked us from auto-closing the popup (see
// closeATCPopup / openATCPopup for why that can happen).
const atcStatus = document.createElement('div');
atcStatus.id = 'ge90-atc-status';
atcStatus.style.display = 'none';
atcStatus.style.fontSize = '11px';
atcStatus.style.fontFamily = 'Arial, sans-serif';
atcStatus.style.color = '#ffcf6b';
atcStatus.style.marginTop = '2px';
atcStatus.style.lineHeight = '1.3';
atcContainer.appendChild(atcStatus);
// Add to GE90 panel
wrap.appendChild(atcContainer);
// ---------------------------------------------------------------
// V1 Speed field (placed directly under the Live ATC area)
// ---------------------------------------------------------------
const v1Container = document.createElement('div');
v1Container.style.display = 'flex';
v1Container.style.flexDirection = 'column';
v1Container.style.gap = '6px';
v1Container.style.marginTop = '10px';
const v1Title = document.createElement('div');
v1Title.textContent = 'V1 Speed';
v1Title.style.color = 'white';
v1Title.style.fontSize = '12px';
v1Title.style.fontFamily = 'Arial, sans-serif';
v1Title.style.fontWeight = 'bold';
v1Container.appendChild(v1Title);
const v1Input = document.createElement('input');
v1Input.id = 'v1speed';
v1Input.placeholder = 'V1 Speed (kts)';
v1Input.title = "Type in your calculated V1 speed here";
v1Input.style.width = '160px';
v1Input.style.padding = '3px 6px';
v1Input.style.borderRadius = '6px';
v1Input.style.border = '1px solid rgba(255,255,255,0.3)';
v1Input.style.background = 'rgba(255,255,255,0.12)';
v1Input.style.color = 'white';
v1Input.style.fontSize = '12px';
v1Input.style.fontFamily = 'Arial, sans-serif';
v1Container.appendChild(v1Input);
wrap.appendChild(v1Container);
      document.body.appendChild(wrap);
    }
    // Volume label stays synced to whichever pack is active; the panel itself is visible on any aircraft, but the volume slider only makes sense with an active pack.
    function updateSliderVisibilityAndLabel(){
      const wrap = document.getElementById('ge90-volume-wrap');
      const label = document.getElementById('ge90-volume-label');
      const slider = document.getElementById('ge90-volume-slider');
      const divider = document.getElementById('ge90-volume-divider');
      const acRow = document.getElementById('ge90-ac-row');
      const info = activePackInfo();
      if (!wrap || !label || !slider) return;
      wrap.style.display = window._GE_sliderVisible ? 'block' : 'none';
      if (info) {
        label.textContent = info.label;
        label.style.display = '';
        slider.style.display = 'block';
        if (divider) divider.style.display = '';
        // Air Conditioning shares this pack's audio fencing/allow-list,
        // so it's only meaningful on the six aircraft with a custom sound
        // pack active — same condition as the volume slider above it.
        if (acRow) acRow.style.display = 'flex';
      } else {
        label.style.display = 'none';
        slider.style.display = 'none';
        if (divider) divider.style.display = 'none';
        if (acRow) acRow.style.display = 'none';
      }
    }
    // Smooth final master gain — applied to whichever pack is active
    let lastUpdate = performance.now();
    function smoothFinalVolume(){
      const now = performance.now();
      const dt = (now - lastUpdate) / 1000;
      lastUpdate = now;
      const smoothing = 1 - Math.exp(-2.0 * dt);
      const info = activePackInfo();
      if (!info) return;
      const finalMaster = window['_' + info.prefix + '_finalMaster'];
      if (!finalMaster) return;
      const target = window._GE_customVolume;
      const g = finalMaster.gain;
      g.value += (target - g.value) * smoothing;
    }
    setInterval(smoothFinalVolume, 50);
    setInterval(updateSliderVisibilityAndLabel, 250);
    setTimeout(createVolumeSlider, 1500);
    // The configurable hide-key toggles the whole panel, and also captures a new shortcut when the user clicks 'Hide Panel Key'. Typing in inputs elsewhere is ignored.
    function bindToggle(){
      if (window._GE_toggleBound) return;
      window._GE_toggleBound = true;
      window.addEventListener('keydown', function(e){
        try {
          const t = e.target;
          if (t && (t.tagName === 'INPUT' || t.tagName === 'TEXTAREA' || t.isContentEditable)) return;
          // Capturing a new hide-key shortcut: consume this keypress to
          // set (or, on Escape, cancel setting) it instead of treating it
          // as anything else.
          if (window._GE_listeningForHideKey) {
            e.preventDefault();
            e.stopPropagation();
            window._GE_listeningForHideKey = false;
            const btn = document.getElementById('ge90-hidekey-btn');
            if (e.code !== 'Escape') {
              window._GE_hideKey = e.code;
              try { localStorage.setItem('GE90_hideKey', e.code); } catch(err){}
            }
            if (btn) {
              btn.textContent = prettyKeyLabel(window._GE_hideKey);
              btn.style.background = 'rgba(255,255,255,0.12)';
            }
            return;
          }
          if (e.code !== window._GE_hideKey) return;
          if (!document.getElementById('ge90-volume-wrap')) createVolumeSlider();
          window._GE_sliderVisible = !window._GE_sliderVisible;
          updateSliderVisibilityAndLabel();
        } catch(err){
          console.warn('[GE90 Ultimate] toggle error', err);
        }
      }, { capture: true });
    }
    bindToggle();
  })();
})();
setTimeout((function() {
    'use strict';
    window.soundsToggleKey = ""; //CHANGE THIS LETTER TO CHANGE THE KEYBOARD SHORTCUT TO TOGGLE THE SOUNDS.
    window.soundsOn = true; //This decides whether callouts are on by default or off by default.
    // COMPATIBILITY LAYER: this pack mutes unrecognized <audio> elements while active; GPWS Audio objects are tagged so they're never caught by that guard.
    function geTagAudio(a) {
        try { a.dataset.geofsAllow = 'true'; a.dataset.gpwsAllow = 'true'; } catch (e) {}
        return a;
    }
    // SMOOTH GPWS ALARM LOOPS: masterA/bankAngle/overspeed are plain <audio
    // loop> elements used elsewhere via play()/pause()/currentTime, so rather
    // than restructure them, this decodes once, bakes the same equal-power
    // crossfade into the tail, re-encodes as a WAV Blob, and swaps it in as
    // the element's src — purely additive; falls back to the original source
    // unchanged if anything fails.
    function audioBufferToWavBlob(buffer) {
        var numCh = buffer.numberOfChannels;
        var sr = buffer.sampleRate;
        var len = buffer.length;
        var blockAlign = numCh * 2;
        var dataSize = len * blockAlign;
        var ab = new ArrayBuffer(44 + dataSize);
        var view = new DataView(ab);
        function writeStr(offset, str) {
            for (var i = 0; i < str.length; i++) view.setUint8(offset + i, str.charCodeAt(i));
        }
        writeStr(0, 'RIFF');
        view.setUint32(4, 36 + dataSize, true);
        writeStr(8, 'WAVE');
        writeStr(12, 'fmt ');
        view.setUint32(16, 16, true);
        view.setUint16(20, 1, true); // PCM
        view.setUint16(22, numCh, true);
        view.setUint32(24, sr, true);
        view.setUint32(28, sr * blockAlign, true);
        view.setUint16(32, blockAlign, true);
        view.setUint16(34, 16, true);
        writeStr(36, 'data');
        view.setUint32(40, dataSize, true);
        var channels = [];
        for (var ch = 0; ch < numCh; ch++) channels.push(buffer.getChannelData(ch));
        var offset = 44;
        for (var i2 = 0; i2 < len; i2++) {
            for (var ch2 = 0; ch2 < numCh; ch2++) {
                var s = Math.max(-1, Math.min(1, channels[ch2][i2]));
                s = s < 0 ? s * 0x8000 : s * 0x7FFF;
                view.setInt16(offset, s, true);
                offset += 2;
            }
        }
        return new Blob([ab], { type: 'audio/wav' });
    }
    var _gpwsSmoothCtx = null;
    async function smoothAlarmLoop(audioEl, url, fadeSec) {
        try {
            var AC = window.AudioContext || window.webkitAudioContext;
            if (!AC) return;
            if (!_gpwsSmoothCtx) _gpwsSmoothCtx = new AC();
            var r = await fetch(url, { mode: 'cors' });
            var ab = await r.arrayBuffer();
            var decoded = await _gpwsSmoothCtx.decodeAudioData(ab);
            var smoothed = (typeof window._GE_seamlessLoopBuffer === 'function')
                ? window._GE_seamlessLoopBuffer(_gpwsSmoothCtx, decoded, fadeSec)
                : decoded;
            var blob = audioBufferToWavBlob(smoothed);
            var blobUrl = URL.createObjectURL(blob);
            var applySwap = function(){ try { audioEl.src = blobUrl; } catch(e){} };
            // Never swap out from under an alarm that's actively sounding —
            // wait for the next time it's paused (GPWS pauses/resets these
            // constantly as conditions clear) so there's no audible hiccup.
            if (audioEl.paused) {
                applySwap();
            } else {
                audioEl.addEventListener('pause', applySwap, { once: true });
            }
        } catch (e) {
            // Silent fallback: original source keeps playing as before.
        }
    }
    // Shared view-type detection mirrors the Sound Pack Ultimate's own getViewType() so both scripts agree on the current camera view.
    function getGEViewType() {
        try {
            var cam = window.geofs && window.geofs.camera;
            if (!cam) return "exterior";
            var s = cam.currentModeName || cam.currentView || (cam.currentDefinition && cam.currentDefinition.name) || cam.mode || "";
            s = String(s).toLowerCase();
            if (!s) return "exterior";
            if (
                s.includes("cockpit") ||
                s.includes("jumpseat") ||
                s.includes("jump seat") ||
                s.includes("virtual cockpit") ||
                s.includes("2d cockpit") ||
                s.includes("pilot") ||
                s.includes("interior") ||
                s.includes("vc")
            ) return "cockpit";
            if (
                (s.includes("left wing") || s.includes("right wing") || s.includes("left") || s.includes("right")) &&
                (s.includes("cfm") || s.includes("iaev") || s.includes("v2500") || s.includes("engine"))
            ) return "wingEngine";
            if (
                s.includes("wing 2") ||
                s.includes("wing2") ||
                s.includes("wing view") ||
                s.includes("external wing") ||
                s.includes("left wing") ||
                s.includes("right wing")
            ) return "wing2";
            return "exterior";
        } catch (e) {
            return "exterior";
        }
    }
    // Views in which GPWS callouts should play 50% quieter (half volume),
    // as defined by the GE90 Sound Pack Ultimate's view categories.
    window.gpwsQuietViews = ["cockpit", "wing2", "wingEngine"];
    window.gpwsQuietVolume = 0.40; // 50% quieter
    window.gpwsNormalVolume =1.0;
    // Applies correct per-view volume to every GPWS audio object each tick (cheap; needed since Boeing/default swap recreates them). Scaled further by the active pack's volume slider, if any.
    function applyGPWSCalloutVolume() {
        var viewType = getGEViewType();
        var vol = (window.gpwsQuietViews.indexOf(viewType) !== -1) ? window.gpwsQuietVolume : window.gpwsNormalVolume;
        var packActive = false;
        try {
            packActive = !!(window._GEPacks && Object.values(window._GEPacks).some(function(v){ return v === true; }));
        } catch (e) {}
        if (packActive) {
            var sliderVol = (typeof window._GE_customVolume === 'number' && !isNaN(window._GE_customVolume)) ? window._GE_customVolume : 1.0;
            vol = vol * sliderVol;
        }
        var audios = [
            window.a2500, window.a2000, window.a1000, window.a500, window.a400,
            window.a300, window.a200, window.a100, window.a50, window.a40,
            window.a30, window.a20, window.a10, window.aRetard, window.a5,
            window.stall, window.glideSlope, window.tooLowFlaps, window.tooLowGear,
            window.apDisconnect, window.minimumBaro, window.dontSink,
            window.masterA, window.bankAngle, window.overspeed, window.v1Callout, window.v1CalloutAirbus
        ];
        for (var idx = 0; idx < audios.length; idx++) {
            if (!audios[idx]) continue;
            // V1 gets a 15% boost on top of the shared callout volume so it stands out during
            // the takeoff roll; the Airbus V1 callout gets a further 10% on top of that
            // (so ~26.5% over the shared callout volume). Clamped to 1.0 since
            // HTMLMediaElement.volume can't exceed that.
            if (audios[idx] === window.v1CalloutAirbus) {
                audios[idx].volume = Math.min(1, vol * 1.15 * 1.10);
            } else if (audios[idx] === window.v1Callout) {
                audios[idx].volume = Math.min(1, vol * 1.15);
            } else {
                audios[idx].volume = vol;
            }
        }
    }
    window.a2500 = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/2500.wav'));
    window.a2000 = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/2000.wav'));
    window.a1000 = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/1000.wav'));
    window.a500 = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/500.wav'));
    window.a400 = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/400.wav'));
    window.a300 = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/300.wav'));
    window.a200 = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/200.wav'));
    window.a100 = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/100.wav'));
    window.a50 = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/50.wav'));
    window.a40 = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/40.wav'));
    window.a30 = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/30.wav'));
    window.a20 = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/20.wav'));
    window.a10 = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/10.wav'));
    window.aRetard = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/retard.wav'));
    window.a5 = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/5.wav'));
    window.stall = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/stall.wav'));
    window.glideSlope = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/glideslope.wav'));
    window.tooLowFlaps = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/too-low_flaps.wav'));
    window.tooLowGear = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/too-low_gear.wav'));
    window.apDisconnect = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/ap-disconnect.wav'));
    window.minimumBaro = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/minimum.wav'));
    window.dontSink = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/dont-sink.wav'));
    window.masterA = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/masterAlarm.wav'));
    window.bankAngle = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/bank-angle.wav'));
    window.overspeed = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/overspeed.wav'));
    smoothAlarmLoop(window.masterA, 'https://tylerbmusic.github.io/GPWS-files_geofs/masterAlarm.wav', 0.12);
    smoothAlarmLoop(window.bankAngle, 'https://tylerbmusic.github.io/GPWS-files_geofs/bank-angle.wav', 0.12);
    smoothAlarmLoop(window.overspeed, 'https://tylerbmusic.github.io/GPWS-files_geofs/overspeed.wav', 0.12);
    window.v1Callout = geTagAudio(new Audio('https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Sound-Pack-Ultimate-/main/V1.wav'));
    // Airbus family (a320ceo/a320neo/a330ceo/a340/a350/a380) gets a distinct
    // V1 callout; B737/B777 (and unrecognized) keep V1.wav. This GPWS block
    // runs in its own setTimeout scope, separate from the GE90 IIFE, so it
    // needs its own aircraft-ID lookup.
    window.v1CalloutAirbus = geTagAudio(new Audio('https://raw.githubusercontent.com/ChristianPilotAlex003/GeoFS-Sound-Pack-Ultimate-/main/Airbus%20V1.wav'));
    window._GE_AIRBUS_V1_IDS = [
        '5847','2871','2865','242','4646',   // a320neo
        '5156','2879','3534','3011','5086','2870', // a320ceo
        '244','2856','6012',                 // a330ceo
        '4631',                              // a330neo (a330-900)
        '6006','2153','5998',                // a340
        '24','2973','239',                   // a350
        '10'                                 // a380
    ];
    window._GE_getV1CalloutAudio = window._GE_getV1CalloutAudio || function(){
        try {
            var id = String((window.geofs && window.geofs.aircraft && window.geofs.aircraft.instance && window.geofs.aircraft.instance.id) ?? '');
            if (id && window._GE_AIRBUS_V1_IDS.indexOf(id) !== -1) {
                return window.v1CalloutAirbus;
            }
        } catch(e){}
        return window.v1Callout; // default: B737/B777/unrecognized
    };
    window.justPaused = false;
    window.masterA.loop = true;
    window.bankAngle.loop = true;
    window.overspeed.loop = true;
    window.iminimums = false;
    window.iv1 = false;
    window.i2500 = false;
    window.i2000 = false;
    window.i1000 = false;
    window.i500 = false;
    window.i400 = false;
    window.i300 = false;
    window.i200 = false;
    window.i100 = false;
    window.i50 = false;
    window.i40 = false;
    window.i30 = false;
    window.i20 = false;
    window.i10 = false;
    window.i7 = false;
    window.i5 = false;
    window.gpwsRefreshRate = 100;
    window.willTheDoorFallOff = false;
    window.didAWheelFall = false;
    function isInRange(i, a, vs) {
        if (i >= 100) {
            if ((i <= a+10) && (i >= a-10)) {
                return true;
            }
        } else if (i >= 10) {
            if ((i < a+4) && (i > a-4)) {
                return true;
            }
        } else {
            if (i <= a+1 && i >= a-1) {
                return true;
            }
        }
        return false;
    }
    window.wasAPOn = false;
    var flightDataElement = document.getElementById('flightDataDisplay1');
    if (!flightDataElement) {
        var bottomDiv = document.getElementsByClassName('geofs-ui-bottom')[0];
        flightDataElement = document.createElement('div');
        flightDataElement.id = 'flightDataDisplay1';
        flightDataElement.classList = 'mdl-button';
        bottomDiv.appendChild(flightDataElement);
    }
    flightDataElement.innerHTML = `
                <input style="background: 0 0; border: none; border-radius: 2px; color: #000; display: inline-block; padding: 0 8px;" placeholder="Minimums (Baro)" id="minimums">
            `;
    function updateGPWS() {
        // Check if geofs.animation.values is available
        if (typeof geofs.animation.values != 'undefined' && !geofs.isPaused()) {
            if (window.justPaused) {
                window.justPaused = false;
            }
            window.willTheDoorFallOff = geofs.aircraft.instance.aircraftRecord.name.includes("Boeing");
            if (window.willTheDoorFallOff && !window.didAWheelFall) {
                window.a2500 = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/b2500.wav'));
                window.a2000 = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/b2000.wav'));
                window.a1000 = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/b1000.wav'));
                window.a500 = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/b500.wav'));
                window.a400 = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/b400.wav'));
                window.a300 = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/b300.wav'));
                window.a200 = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/b200.wav'));
                window.a100 = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/b100.wav'));
                window.a50 = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/b50.wav'));
                window.a40 = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/b40.wav'));
                window.a30 = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/b30.wav'));
                window.a20 = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/b20.wav'));
                window.a10 = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/b10.wav'));
                window.a5 = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/b5.wav'));
                window.stall = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/bstall.wav'));
                window.glideSlope = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/bglideslope.wav'));
                window.tooLowFlaps = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/btoo-low_flaps.wav'));
                window.tooLowGear = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/btoo-low_gear.wav'));
                window.apDisconnect = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/bap-disconnect.wav'));
                window.minimumBaro = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/bminimums.wav'));
                window.dontSink = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/bdont-sink.wav'));
                window.masterA = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/bmasterAlarm.wav'));
                window.bankAngle = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/bbank-angle.wav'));
                window.overspeed = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/boverspeed.wav'));
                smoothAlarmLoop(window.masterA, 'https://tylerbmusic.github.io/GPWS-files_geofs/bmasterAlarm.wav', 0.12);
                smoothAlarmLoop(window.bankAngle, 'https://tylerbmusic.github.io/GPWS-files_geofs/bbank-angle.wav', 0.12);
                smoothAlarmLoop(window.overspeed, 'https://tylerbmusic.github.io/GPWS-files_geofs/boverspeed.wav', 0.12);
                window.masterA.loop = true;
                window.bankAngle.loop = true;
                window.overspeed.loop = true;
            } else if (!window.willTheDoorFallOff && window.didAWheelFall) {
                window.a2500 = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/2500.wav'));
                window.a2000 = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/2000.wav'));
                window.a1000 = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/1000.wav'));
                window.a500 = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/500.wav'));
                window.a400 = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/400.wav'));
                window.a300 = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/300.wav'));
                window.a200 = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/200.wav'));
                window.a100 = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/100.wav'));
                window.a50 = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/50.wav'));
                window.a40 = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/40.wav'));
                window.a30 = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/30.wav'));
                window.a20 = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/20.wav'));
                window.a10 = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/10.wav'));
                window.a5 = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/5.wav'));
                window.stall = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/stall.wav'));
                window.glideSlope = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/glideslope.wav'));
                window.tooLowFlaps = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/too-low_flaps.wav'));
                window.tooLowGear = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/too-low_gear.wav'));
                window.apDisconnect = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/ap-disconnect.wav'));
                window.minimumBaro = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/minimum.wav'));
                window.dontSink = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/dont-sink.wav'));
                window.masterA = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/masterAlarm.wav'));
                window.bankAngle = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/bank-angle.wav'));
                window.overspeed = geTagAudio(new Audio('https://tylerbmusic.github.io/GPWS-files_geofs/overspeed.wav'));
                smoothAlarmLoop(window.masterA, 'https://tylerbmusic.github.io/GPWS-files_geofs/masterAlarm.wav', 0.12);
                smoothAlarmLoop(window.bankAngle, 'https://tylerbmusic.github.io/GPWS-files_geofs/bank-angle.wav', 0.12);
                smoothAlarmLoop(window.overspeed, 'https://tylerbmusic.github.io/GPWS-files_geofs/overspeed.wav', 0.12);
                window.masterA.loop = true;
                window.bankAngle.loop = true;
                window.overspeed.loop = true;
            }
            // Keep callout volume in sync with the current camera view
            // (50% quieter in cockpit/jumpseat/wing2/wingEngine views).
            applyGPWSCalloutVolume();
            // Retrieve and format the required values
            var minimum = ((document.getElementById("minimums") !== null) && document.getElementById("minimums").value !== undefined) ? Number(document.getElementById("minimums").value) : undefined;
            var v1Target = ((document.getElementById("v1speed") !== null) && document.getElementById("v1speed").value !== "") ? Number(document.getElementById("v1speed").value) : undefined;
            // --- GeoFS 4.0 ground-elevation source fix ---
            // groundElevationFeet can be stale/wrong at airports with flattened
            // terrain (e.g. JFK) though fine elsewhere. getGroundAltitude(lat, lon)
            // raycasts the actual rendered mesh instead (same as the radio
            // altimeter), used as the primary source; falls back to
            // groundElevationFeet (jump-clamped) only if the raycast fails.
            var groundElevFt;
            try {
                var _geGroundHit = geofs.getGroundAltitude(geofs.aircraft.instance.llaLocation[0], geofs.aircraft.instance.llaLocation[1]);
                var _geGroundAltM = (_geGroundHit && _geGroundHit.location) ? _geGroundHit.location[2] : undefined;
                groundElevFt = (_geGroundAltM !== undefined && !Number.isNaN(_geGroundAltM)) ? (_geGroundAltM * 3.2808399) : undefined;
            } catch (_geGroundErr) {
                groundElevFt = undefined;
            }
            if (groundElevFt === undefined) {
                // Fallback: cached animation-loop terrain sample, glitch-clamped against the last good reading.
                var rawGroundElevFt = geofs.animation.values.groundElevationFeet;
                if (rawGroundElevFt === undefined || rawGroundElevFt === null || Number.isNaN(rawGroundElevFt)) {
                    groundElevFt = window._geLastGoodGroundElevFt;
                } else if (window._geLastGoodGroundElevFt === undefined) {
                    groundElevFt = rawGroundElevFt;
                } else {
                    var _geElevJumpFt = rawGroundElevFt - window._geLastGoodGroundElevFt;
                    var _geMaxStepFt = 75;
                    groundElevFt = (Math.abs(_geElevJumpFt) > _geMaxStepFt)
                        ? window._geLastGoodGroundElevFt + Math.sign(_geElevJumpFt) * _geMaxStepFt
                        : rawGroundElevFt;
                }
            }
            if (groundElevFt !== undefined && !Number.isNaN(groundElevFt)) window._geLastGoodGroundElevFt = groundElevFt;
            // --- Self-calibrating ground-reference bias correction ---
            // Some GeoFS builds carry a small constant AGL bias (e.g. ~15ft on
            // the ground), which can suppress the 20/10/5 callouts. Rather than
            // hardcode an offset, this nudges a running bias estimate toward 0
            // whenever groundContact is true and subtracts it from every AGL
            // reading — a no-op on builds with no bias.
            var rawAgl = (geofs.animation.values.altitude !== undefined && groundElevFt !== undefined) ? ((geofs.animation.values.altitude - groundElevFt) + (geofs.aircraft.instance.collisionPoints[geofs.aircraft.instance.collisionPoints.length - 2].worldPosition[2]*3.2808399)) : undefined;
            if (rawAgl !== undefined && geofs.aircraft.instance.groundContact) {
                if (window._geAglGroundBiasFt === undefined) window._geAglGroundBiasFt = rawAgl; // seed instantly so ground callouts are right from the first landing/spawn
                var _geBiasErr = rawAgl - window._geAglGroundBiasFt;
                var _geBiasStep = Math.max(-2, Math.min(2, _geBiasErr * 0.1)); // slow-converging average, not a snap
                window._geAglGroundBiasFt = Math.max(-30, Math.min(30, window._geAglGroundBiasFt + _geBiasStep)); // sanity-capped; this is a small-bias fix, not a terrain fallback
            }
            var agl = (rawAgl !== undefined) ? Math.round(rawAgl - (window._geAglGroundBiasFt || 0)) : 'N/A';
            var verticalSpeed = geofs.animation.values.verticalSpeed !== undefined ? Math.round(geofs.animation.values.verticalSpeed) : 'N/A';
            //Glideslope calculation
            var glideslope;
            if (geofs.animation.getValue("NAV1Direction") && (geofs.animation.getValue("NAV1Distance") !== geofs.runways.getNearestRunway([geofs.nav.units.NAV1.navaid.lat,geofs.nav.units.NAV1.navaid.lon,0]).lengthMeters*0.185)) { //The second part to the if statement prevents the divide by 0 error.
                glideslope = (geofs.animation.getValue("NAV1Direction") === "to") ? Number((Math.atan(((geofs.animation.values.altitude/3.2808399+(geofs.aircraft.instance.collisionPoints[geofs.aircraft.instance.collisionPoints.length - 2].worldPosition[2]+0.1))-geofs.nav.units.NAV1.navaid.elevation) / (geofs.animation.getValue("NAV1Distance")+geofs.runways.getNearestRunway([geofs.nav.units.NAV1.navaid.lat,geofs.nav.units.NAV1.navaid.lon,0]).lengthMeters*0.185))*RAD_TO_DEGREES).toFixed(1)) : Number((Math.atan(((geofs.animation.values.altitude/3.2808399+(geofs.aircraft.instance.collisionPoints[geofs.aircraft.instance.collisionPoints.length - 2].worldPosition[2]+0.1))-geofs.nav.units.NAV1.navaid.elevation) / Math.abs(geofs.animation.getValue("NAV1Distance")-geofs.runways.getNearestRunway([geofs.nav.units.NAV1.navaid.lat,geofs.nav.units.NAV1.navaid.lon,0]).lengthMeters*0.185))*RAD_TO_DEGREES).toFixed(1));
            } else {
                glideslope = undefined;
            } //End Glideslope calculation
            if (window.soundsOn) {
                if (((geofs.aircraft.instance.stalling && !geofs.aircraft.instance.groundContact) || (geofs.nav.units.NAV1.navaid !== null && (agl > 100 && (glideslope < (geofs.nav.units.NAV1.navaid.slope - 1.5) || (glideslope > geofs.nav.units.NAV1.navaid.slope + 2)))) || (!geofs.aircraft.instance.groundContact && agl < 300 && (geofs.aircraft.instance.definition.gearTravelTime !== undefined) && (geofs.animation.values.gearPosition >= 0.5)) || (!geofs.aircraft.instance.groundContact && agl < 500 && (geofs.animation.values.flapsSteps !== undefined) && (geofs.animation.values.flapsPosition == 0) && window.tooLowGear.paused) || (!geofs.aircraft.instance.groundContact && agl < 300 && geofs.animation.values.throttle > 0.95 && verticalSpeed <= 0) || (Math.abs(geofs.aircraft.instance.animationValue.aroll) > 45)) && window.masterA.paused) {
                    window.masterA.play();
                } else if (!((geofs.aircraft.instance.stalling && !geofs.aircraft.instance.groundContact) || (geofs.nav.units.NAV1.navaid !== null && (agl > 100 && (glideslope < (geofs.nav.units.NAV1.navaid.slope - 1.5) || (glideslope > geofs.nav.units.NAV1.navaid.slope + 2)))) || (!geofs.aircraft.instance.groundContact && agl < 300 && (geofs.aircraft.instance.definition.gearTravelTime !== undefined) && (geofs.animation.values.gearPosition >= 0.5)) || (!geofs.aircraft.instance.groundContact && agl < 500 && (geofs.animation.values.flapsSteps !== undefined) && (geofs.animation.values.flapsPosition == 0) && window.tooLowGear.paused) || (!geofs.aircraft.instance.groundContact && agl < 300 && geofs.animation.values.throttle > 0.95 && verticalSpeed <= 0) || (Math.abs(geofs.aircraft.instance.animationValue.aroll) > 45)) && !window.masterA.paused) {
                    window.masterA.pause();
                }
                if (Math.abs(geofs.aircraft.instance.animationValue.aroll) > 45 && window.bankAngle.paused) {
                    window.bankAngle.play();
                } else if (!(Math.abs(geofs.aircraft.instance.animationValue.aroll) > 45) && !window.bankAngle.paused) {
                    window.bankAngle.pause();
                }
                var _geOverspeedLimit = (typeof geofs.animation.values.VNO === 'number' && geofs.animation.values.VNO > 0) ? geofs.animation.values.VNO : 350; //Falls back to 350kt if the aircraft doesn't define VNO
                if (geofs.animation.values.kias > _geOverspeedLimit && window.overspeed.paused) { //Overspeed (was previously never triggered anywhere in the script)
                    window.overspeed.play();
                } else if (!window.overspeed.paused && geofs.animation.values.kias <= _geOverspeedLimit) {
                    window.overspeed.pause();
                }
                if (geofs.aircraft.instance.stalling && !geofs.aircraft.instance.groundContact && window.stall.paused) { //Stall
                    window.stall.play();
                } else if (!window.stall.paused && !geofs.aircraft.instance.stalling) {
                    window.stall.pause();
                }
                if (geofs.nav.units.NAV1.navaid !== null && (agl > 100 && (glideslope < (geofs.nav.units.NAV1.navaid.slope - 1.5) || (glideslope > geofs.nav.units.NAV1.navaid.slope + 2)) && window.glideSlope.paused)) { //Glideslope
                    window.glideSlope.play();
                }
                if (!geofs.aircraft.instance.groundContact && agl < 300 && (geofs.aircraft.instance.definition.gearTravelTime !== undefined) && (geofs.animation.values.gearPosition >= 0.5) && window.tooLowGear.paused) { //Too Low - Gear (This warning takes priority over the Too Low - Flaps warning)
                    window.tooLowGear.play();
                }
                if (!geofs.aircraft.instance.groundContact && agl < 500 && (geofs.animation.values.flapsSteps !== undefined) && (geofs.animation.values.flapsPosition == 0) && window.tooLowGear.paused && window.tooLowFlaps.paused) { //Too Low - Flaps
                    window.tooLowFlaps.play();
                }
                if (!geofs.autopilot.on && window.wasAPOn) { //Autopilot Disconnect
                    window.apDisconnect.play();
                }
                // V1 callout: fires during takeoff roll once past typed V1 speed,
                // with flaps out (flapsPosition > 0.01) and still accelerating
                // (measured via performance.now() each tick, not edge-triggered, so
                // a glitch tick can't cause a miss). Plays V1.wav for B737/B777/
                // unrecognized aircraft, or the Airbus callout for a320ceo/a320neo/
                // a330ceo/a340/a350/a380 (see _GE_getV1CalloutAudio()/
                // _GE_AIRBUS_V1_IDS). Volume-gated outside cockpit like other GPWS
                // callouts. No longer requires flapsSteps.
                if (v1Target !== undefined && !isNaN(v1Target) && v1Target > 0) {
                    var _geKiasNow = geofs.animation.values.kias;
                    var _geNowTsV1 = performance.now();
                    var _geHasFlapsOut = (typeof geofs.animation.values.flapsPosition === 'number') && (geofs.animation.values.flapsPosition > 0.01);
                    var _geV1Accel = 0; // knots per second
                    if (typeof window._geLastKiasForV1Accel === 'number' && typeof window._geLastTimeForV1Accel === 'number') {
                        var _geDtV1 = (_geNowTsV1 - window._geLastTimeForV1Accel) / 1000;
                        if (_geDtV1 > 0 && typeof _geKiasNow === 'number') {
                            _geV1Accel = (_geKiasNow - window._geLastKiasForV1Accel) / _geDtV1;
                        }
                    }
                    window._geLastKiasForV1Accel = _geKiasNow;
                    window._geLastTimeForV1Accel = _geNowTsV1;
                    var V1_MIN_ACCEL_KTS_PER_SEC = 0.15; // small threshold: just enough to confirm the plane is still actually speeding up
                                        if (geofs.aircraft.instance.groundContact &&
                        typeof _geKiasNow === 'number' && _geKiasNow >= v1Target && _geHasFlapsOut &&
                        _geV1Accel >= V1_MIN_ACCEL_KTS_PER_SEC && !window.iv1 &&
                        performance.now() >= (window._geV1LockoutUntil || 0)) {
                        // Pick B737/B777 V1.wav or the Airbus-family V1 callout depending on
                        // the currently active aircraft, and make sure the other one is stopped.
                        var _geV1AudioToPlay = window._GE_getV1CalloutAudio();
                        var _geV1AudioOther = (_geV1AudioToPlay === window.v1Callout) ? window.v1CalloutAirbus : window.v1Callout;
                        try { if (_geV1AudioOther && !_geV1AudioOther.paused) { _geV1AudioOther.pause(); _geV1AudioOther.currentTime = 0; } } catch(e){}
                                     _geV1AudioToPlay.play();
                        window.iv1 = true;
                        window._geV1LockoutUntil = performance.now() + 500;
                    }
                    // Re-arm once airborne (next takeoff), or once speed drops back well below V1 while
                    // still on the ground (a rejected takeoff, so it can fire again on the next attempt).
                    if (!geofs.aircraft.instance.groundContact || (typeof _geKiasNow === 'number' && _geKiasNow < v1Target - 5)) {
                        window.iv1 = false;
                        window._geV1LockoutUntil = 0;
                    }
                }
                if (geofs.aircraft.instance.groundContact) {
                    // skipV1=true: V1 requires groundContact===true, so without this the
                    // callout would reset the same tick it starts playing. iv1's re-arm is
                    // handled separately (on liftoff or a rejected takeoff). forceStopGpwsAudio
                    // only runs when a callout flag is set, to avoid iterating all audio
                    // objects every tick while taxiing.
                    if (window.i2500 || window.i2000 || window.i1000 || window.i500 || window.i400 ||
                        window.i300 || window.i200 || window.i100 || window.i50 || window.i40 ||
                        window.i30 || window.i20 || window.i10 || window.i7 || window.i5 || window.iminimums) {
                        window._GE_forceStopGpwsAudio(true);
                    }
                    window.i2500 = window.i2000 = window.i1000 = window.i500 = window.i400 =
                    window.i300 = window.i200 = window.i100 = window.i50 = window.i40 =
                    window.i30 = window.i20 = window.i10 = window.i7 = window.i5 = window.iminimums = false;
                    window.gpwsRefreshRate = 100;
                } else if (verticalSpeed <= 0) {
                    if (!geofs.aircraft.instance.groundContact && agl < 300 && geofs.animation.values.throttle > 0.95 && window.dontSink.paused) { //Don't Sink
                        window.dontSink.play();
                    }
                    if ((minimum !== undefined) && (geofs.animation.values.altitude+2 > minimum && minimum > geofs.animation.values.altitude-2) && !window.iminimums) { //Minimum
                        window.minimumBaro.play();
                        window.iminimums = true;
                    }
                    if (isInRange(2500, agl) && !window.i2500) { //2,500
                        window.a2500.play();
                        window.i2500 = true;
                    }
                    if (isInRange(2000, agl) && !window.i2000) { //2,000
                        window.a2000.play();
                        window.i2000 = true;
                    }
                    if (isInRange(1000, agl) && !window.i1000) { //1,000
                        window.a1000.play();
                        window.i1000 = true;
                    }
                    if (isInRange(500, agl) && !window.i500) { //500
                        window.a500.play();
                        window.i500 = true;
                    }
                    if (isInRange(400, agl) && !window.i400) { //400
                        window.a400.play();
                        window.i400 = true;
                    }
                    if (isInRange(300, agl) && !window.i300) { //300
                        window.a300.play();
                        window.i300 = true;
                    }
                    if (isInRange(200, agl) && !window.i200) { //200
                        window.a200.play();
                        window.i200 = true;
                    }
                    if (isInRange(100, agl) && !window.i100) { //100
                        window.a100.play();
                        window.i100 = true;
                    }
                    if (isInRange(50, agl) && !window.i50) { //50
                        window.a50.play();
                        window.i50 = true;
                    }
                    if (isInRange(40, agl) && !window.i40) { //40
                        window.a40.play();
                        window.i40 = true;
                    }
                    if (isInRange(30, agl) && !window.i30) { //30
                        window.a30.play();
                        window.i30 = true;
                    }
                    if (isInRange(20, agl) && !window.i20) { //20
                        window.a20.play();
                        window.i20 = true;
                    }
                    if (isInRange(10, agl) && !window.i10) { //10
                        window.a10.play();
                        window.i10 = true;
                    }
                    if (!geofs.aircraft.instance.groundContact && ((agl+(geofs.animation.values.verticalSpeed/60)*2) <= 1.0) && !window.i7) { //Retard 2 seconds from touchdown
                        window.aRetard.play();
                        window.i7 = true;
                    }
                    if (isInRange(5, agl) && !window.i5) { //5
                        window.a5.play();
                        window.i5 = true;
                    }
                    window.gpwsRefreshRate = 30;
                } else if (verticalSpeed > 0) {
                    if (window.iminimums) {
                        window.iminimums = false;
                    }
                    if (window.i2500) {
                        window.i2500 = false;
                    }
                    if (window.i2000) {
                        window.i2000 = false;
                    }
                    if (window.i1000) {
                        window.i1000 = false;
                    }
                    if (window.i500) {
                        window.i500 = false;
                    }
                    if (window.i400) {
                        window.i400 = false;
                    }
                    if (window.i300) {
                        window.i300 = false;
                    }
                    if (window.i200) {
                        window.i200 = false;
                    }
                    if (window.i100) {
                        window.i100 = false;
                    }
                    if (window.i50) {
                        window.i50 = false;
                    }
                    if (window.i40) {
                        window.i40 = false;
                    }
                    if (window.i30) {
                        window.i30 = false;
                    }
                    if (window.i20) {
                        window.i20 = false;
                    }
                    if (window.i10) {
                        window.i10 = false;
                    }
                    if (window.i7) {
                        window.i7 = false;
                    }
                    if (window.i5) {
                        window.i5 = false;
                    }
                    window.gpwsRefreshRate = 100;
                }
            }
        } else if (geofs.isPaused() && !window.justPaused) {
            window.a2500.pause();
            window.a2000.pause();
            window.a1000.pause();
            window.a500.pause();
            window.a400.pause();
            window.a300.pause();
            window.a200.pause();
            window.a100.pause();
            window.a50.pause();
            window.a40.pause();
            window.a30.pause();
            window.a20.pause();
            window.a10.pause();
            window.aRetard.pause();
            window.a5.pause();
            window.stall.pause();
            window.glideSlope.pause();
            window.tooLowFlaps.pause();
            window.tooLowGear.pause();
            window.apDisconnect.pause();
            window.minimumBaro.pause();
            window.dontSink.pause();
            window.masterA.pause();
            window.bankAngle.pause();
            window.overspeed.pause();
            window.v1Callout.pause();
            window.v1CalloutAirbus.pause();
            window.justPaused = true;
        }
        window.wasAPOn = geofs.autopilot.on;
        window.didAWheelFall = window.willTheDoorFallOff;
    }
    // Update flight data display (rate adapts: 30ms while descending, 100ms otherwise).
    // Uses a self-rescheduling setTimeout instead of setInterval because setInterval
    // locks in its delay at creation time and never re-reads window.gpwsRefreshRate,
    // which made the adaptive rate a no-op.
    (function scheduleGPWS(){
        updateGPWS();
        window._geGpwsTimer = setTimeout(scheduleGPWS, window.gpwsRefreshRate);
    })();
    document.addEventListener('keydown', function(event) {
                if (event.key === window.soundsToggleKey) {
                    window.soundsOn = !window.soundsOn;
                }
        });
}), 8000);