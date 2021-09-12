---
title: roam/js
---

- [[roam-toc]]
	 - ```clojure
(ns toc
  (:require
   [clojure.string]
   [roam.block :as block]
   [roam.datascript :as d]
   [roam.datascript.reactive :as dr]
   [roam.util :as u]))

(defn ancestor-block-uids [uid]
  (d/q '[:find [?puid ...]
         :in $ ?cuid
         :where
         [?c :block/uid ?cuid]
         [?c :block/parents ?p]
         [?p :block/string]
         [?p :block/uid ?puid]]
       uid))

(defn block-html-el [uid]
  (some (fn [x]
          (when (clojure.string/ends-with? (.-id x) uid) x))
        (.getElementsByClassName js/document "rm-block__input")))

(defn try-scroll-into-view [uid]
  (when-some [el (block-html-el uid)]
      (.scrollIntoView el)))

(defn scroll-to-block [uid]
  (doseq [parent (ancestor-block-uids uid)]
    (block/update
     {:block {:uid parent :open true}}))
  (when-not (try-scroll-into-view uid)
    (js/setTimeout #(try-scroll-into-view uid) 300)))

(defn flatten-block [acc block]
  (reduce flatten-block
          (conj acc (dissoc block :block/children))
          (sort-by :block/order (:block/children block))))

(defn block->table-entry [block]
  [:div {:style {:margin-left (str (dec (:block/heading block)) "em")
                 :font-size (str (* 0.25 (- 7 (:block/heading block)))
                                 "em")}}
   [:a {:on-click #(scroll-to-block (:block/uid block))}
    (u/parse (:block/string block))]])

(defn component [{uid :block-uid}]
  (let [block @(dr/q '[:find (pull ?p [:block/string
                                       :block/uid
                                       :block/heading
                                       :block/order
                                       {:block/children ...}]) .
                       :in $ ?uid
                       :where
                       [?e :block/uid ?uid]
                       [?e :block/parents ?p]
                       [?p :node/title]]
                     uid)]
    (into [:div]
          (comp
           (filter #(-> % :block/heading pos?))
           (map block->table-entry))
          (flatten-block [] block))))
```
id:: b95bf7e2-b351-47d5-b7c1-e5bc8f528424

- [[roam/js/预览]]
	 - {{[[roam/js]]}}
		 - ```javascript
// ==UserScript==
// @name         Roam live preview
// @namespace    http://tampermonkey.net/
// @version      0.2
// @description  Live preview for page refs on hover
// @author       palashkaria
// @match        https://roamresearch.com
// @downloadURL  https://raw.github.com/palashkaria/roam-modifiers/master/roam-live-preview.user.js
// @grant        none
// ==/UserScript==

(function () {
  'use strict';
  const runInPageContext = (method, ...args) => {
    // will be parsed as a function object.
    console.log(method);
    console.log(args);
    const stringifiedMethod = method.toString();

    const stringifiedArgs = JSON.stringify(args);

    const scriptContent = `document.currentScript.innerHTML = JSON.stringify((${stringifiedMethod})(...${stringifiedArgs}));`;

    const scriptElement = document.createElement('script');
    scriptElement.innerHTML = scriptContent;
    document.documentElement.prepend(scriptElement);

    const result = JSON.parse(scriptElement.innerHTML);
    document.documentElement.removeChild(scriptElement);
    console.log(result);
    return result;
  };
  const getBlockById = (dbId) => {
    const fn = function (dbId) {
      return function (dbId, ...args) {
        return window.roamAlphaAPI.pull(...args, '[*]', dbId);
      };
    };
    return runInPageContext(fn(dbId), dbId);
  };

  const queryFn = (query, ...params) => {
    return runInPageContext(
      function (...args) {
        return window.roamAlphaAPI.q(...args);
      },
      query,
      ...params
    );
  };

  const queryFirst = (query, ...params) => {
    const results = queryFn(query, ...params);
    if (!results || !results[0] || results[0].length < 1) return null;

    return getBlockById(results[0][0]);
  };
  const baseUrl = () => {
    // https://roamresearch.com/#/app/roam-toolkit/page/03-24-2020
    const url = new URL(window.location.href);
    const parts = url.hash.split('/');

    url.hash = parts.slice(0, 3).concat(['page']).join('/');
    return url;
  };
  const getPageByName = (name) => {
    return queryFirst('[:find ?e :in $ ?a :where [?e :node/title ?a]]', name);
  };
  const getPageUrlByName = (name) => {
    const page = getPageByName(name);
    return getPageUrl(page[':block/uid']);
  };
  const getPageUrl = (uid) => {
    return baseUrl().toString() + '/' + uid;
  };

  const createPreviewIframe = () => {
    const iframe = document.createElement('iframe');
    const url = getPageUrl('search');
    const isAdded = (pageUrl) => !!document.querySelector(`[src="${pageUrl}"]`);
    if (isAdded(url)) {
      return;
    }
    iframe.src = url;
    iframe.style.position = 'absolute';
    iframe.style.left = '0';
    iframe.style.top = '0';
    iframe.style.opacity = '0';
    iframe.style.pointerEvents = 'none';

    iframe.style.height = '0';
    iframe.style.width = '0';
    iframe.style.border = '0';
    iframe.style.boxShadow = '0 0 4px 5px rgba(0, 0, 0, 0.2)';
    iframe.style.borderRadius = '4px';
    iframe.id = 'roam-toolkit-preview-iframe';

    const styleNode = document.createElement('style');
    styleNode.innerHTML = `
            .roam-topbar {
                display: none !important;
            }
            .roam-body-main {
                top: 0px !important;
            }
            #buffer {
                display: none !important;
            }
            iframe {
                display: none !important;
           }
        `;
    iframe.onload = (event) => {
      event.target.contentDocument.body.appendChild(styleNode);
    };
    document.body.appendChild(iframe);
    const htmlElement = document.querySelector('html');
    if (htmlElement) {
      // to reset scroll after adding iframe
      htmlElement.scrollTop = 0;
    }
    return iframe;
  };
  const enableLivePreview = () => {
    let hoveredElement = null;
    let popupTimeout = null;
    let popper = null;
    const previewIframe = createPreviewIframe();

    document.addEventListener('mouseover', (e) => {
      const target = e.target;
      const isPageRef = target.classList.contains('rm-page-ref');
      const isPageRefTag = target.classList.contains('rm-page-ref-tag');
      // remove '#' for page tags
      const text = isPageRefTag ? target.innerText.slice(1) : target.innerText;
      if (isPageRef) {
        hoveredElement = target;
        const url = getPageUrlByName(text);
        const isAdded = (pageUrl) =>
          !!document.querySelector(`[src="${pageUrl}"]`);
        const isVisible = (pageUrl) =>
          document.querySelector(`[src="${pageUrl}"]`).style.opacity === '1';
        if ((!isAdded(url) || !isVisible(url)) && previewIframe) {
          previewIframe.src = url;
          previewIframe.style.height = '800px';
          previewIframe.style.width = '1200px';
          previewIframe.style.pointerEvents = 'none';
        }
        if (!popupTimeout) {
          popupTimeout = window.setTimeout(() => {
            if (previewIframe) {
              previewIframe.style.opacity = '1';
              previewIframe.style.pointerEvents = 'all';

              popper = window.Popper.createPopper(target, previewIframe, {
                placement: 'right',
                modifiers: [
                  {
                    name: 'preventOverflow',
                    options: {
                      padding: { top: 48 },
                    },
                  },
                  {
                    name: 'flip',
                    options: {
                      boundary: document.querySelector('#app'),
                    },
                  },
                ],
              });
            }
          }, 300);
        }
      }
    });
    document.addEventListener('mouseout', (e) => {
      const target = e.target;
      const relatedTarget = e.relatedTarget;
      const iframe = document.getElementById('roam-toolkit-preview-iframe');
      if (
        (hoveredElement === target && relatedTarget !== iframe) ||
        (target === iframe && relatedTarget !== hoveredElement) ||
        !document.body.contains(hoveredElement)
      ) {
        hoveredElement = null;
        clearTimeout(popupTimeout);
        popupTimeout = null;
        if (iframe) {
          if (iframe.contentDocument) {
            // scroll to top when removed
            const scrollContainer = iframe.contentDocument.querySelector(
              '.roam-center > div'
            );
            if (scrollContainer) {
              scrollContainer.scrollTop = 0;
            }
          }
          iframe.style.pointerEvents = 'none';
          iframe.style.opacity = '0';
          iframe.style.height = '0';
          iframe.style.width = '0';
        }
        if (popper) {
          popper.destroy();
          popper = null;
        }
      } else {
        console.log('out', target, event);
      }
    });
  };
  var remoteScript = document.createElement('script');
  remoteScript.src =
    'https://unpkg.com/@popperjs/core@2/dist/umd/popper.js?ts=' + +new Date();
  remoteScript.onload = enableLivePreview;
  document.body.appendChild(remoteScript);
})();

````
