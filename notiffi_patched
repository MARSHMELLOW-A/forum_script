var Notiffi = (function () {
  "use strict";

  async function deleteNotification(id, channel) {
    const params = new URLSearchParams();
    params.append("id", id);
    params.append("channel", channel);

    try {
      const response = await fetch("/notif", {
        method: "DELETE",
        headers: { "Content-Type": "application/x-www-form-urlencoded" },
        body: params.toString()
      });

      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }

      const notifEl = document.querySelector(`[data-notif-id="${id}"]`);
      if (notifEl) notifEl.remove();

      return (await response.json()).store;
    } catch (error) {
      console.error("Erreur lors de la suppression :", error);
    }
  }

  async function deleteAllNotifications(ids) {
    const params = new URLSearchParams();
    ids.forEach((id) => params.append("del_notif[]", id));
    params.append("delete_all", "Tout supprimer");

    try {
      const response = await fetch("/profile?mode=editprofile&page_profil=notifications", {
        method: "POST",
        headers: { "Content-Type": "application/x-www-form-urlencoded" },
        body: params.toString()
      });

      if (!response.ok) {
        throw new Error(`HTTP error! status ${response.status}`);
      }

      return true;
    } catch (error) {
      console.error("Erreur lors de la suppression :", error);
      return false;
    }
  }

  async function markNotificationsAsRead(ids) {
    const params = new URLSearchParams();
    ids.forEach((id) => params.append("id[]", id));

    try {
      const response = await fetch("/notif", {
        method: "POST",
        headers: { "Content-Type": "application/x-www-form-urlencoded" },
        body: params.toString()
      });

      if (!response.ok) {
        throw new Error(`HTTP error! status ${response.status}`);
      }

      return true;
    } catch (error) {
      console.error("Erreur lors du marquage comme lu :", error);
      return false;
    }
  }

  async function getUserAvatar(user) {
    const { id, name = "" } = user;
    const parser = new DOMParser();

    const parsedName = parser.parseFromString(name, "text/html");
    const span = parsedName.querySelector("span");
    const username = span
      ? span.textContent.trim()
      : (parsedName.body?.textContent || "").trim();

    if (Notiffi.users[id]) return Notiffi.users[id];

    try {
      const response = await fetch(`/u${id}`);
      if (!response.ok) {
        throw new Error(`Failed to fetch: ${response.statusText}`);
      }

      const html = await response.text();
      const avatarImg = parser
        .parseFromString(html, "text/html")
        .querySelector(`img[alt="${username}"]`);

      const avatar = avatarImg ? `<img loading="lazy" src="${avatarImg.src}" />` : "";

      Notiffi.users[id] = { avatar };
      return Notiffi.users[id];
    } catch (error) {
      console.error(`Error fetching avatar for user ${username || id}`, error);
      return null;
    }
  }

  function renderAward(notification) {
    return notification?.text?.award?.award_image
      ? `<img src="${notification.text.award.award_image}" />`
      : "";
  }

  function bindPanelToggle({ button, panel }) {
    const buttonEl = document.querySelector(button);
    const panelEl = document.querySelector(panel);

    if (!buttonEl || !panelEl) {
      console.error("Notiffi popup : button or panel selector not found.");
      return;
    }

    buttonEl.addEventListener("click", function () {
      buttonEl.classList.toggle("active");
      panelEl.classList.toggle("open");
    });

    document.addEventListener("click", function (event) {
      if (
        !buttonEl.contains(event.target) &&
        !panelEl.contains(event.target) &&
        panelEl.classList.contains("open")
      ) {
        buttonEl.classList.remove("active");
        panelEl.classList.remove("open");
      }
    });
  }

  const filters = {};

  function addFilter(name, fn, priority = 0) {
    if (typeof name !== "string" || typeof fn !== "function") {
      throw new TypeError("Invalid arguments: 'name' must be a string and 'fn' must be a function.");
    }

    filters[name] = filters[name] || [];
    filters[name].push([fn, priority]);
    filters[name].sort((a, b) => a[1] - b[1]);
  }

  function applyFilter(name, value, ...args) {
    return (filters[name] || []).reduce((current, [fn]) => {
      const result = fn(current, ...args);
      return result !== undefined ? result : "";
    }, value);
  }

  function escapeRegExp(str) {
    return str.replace(/[.*+?^${}()|[\]\\]/g, "\\$&");
  }

  addFilter("token", (path, data) => {
    const parts = path.split(".");
    let current = data;

    for (let i = 0; i < parts.length; i++) {
      if (!Object.prototype.hasOwnProperty.call(current, parts[i])) return "";
      current = current[parts[i]];
    }

    return current;
  });

  const userdata = _userdata;
  const storeData = {
    user: {
      name: userdata.username,
      logged_in: Boolean(userdata.session_logged_in),
      level: userdata.user_level,
      id: userdata.user_id,
      posts: userdata.user_posts,
      avatar: userdata.avatar,
      avatar_link: userdata.avatar_link,
      group_color: userdata.groupcolor
    }
  };

  const withStore = (data) => Object.assign({ $store: storeData }, data);

  let potionIndex = 0;
  const potionMap = new Map();
  const tokenCache = new Map();

  function tokenize(template, syntax) {
    if (tokenCache.has(template)) return tokenCache.get(template);

    const regex = new RegExp(
      `${escapeRegExp(syntax.start)}\\s*([!\\/]?)\\s*(${syntax.path})\\s*${escapeRegExp(syntax.end)}`,
      "gi"
    );

    let match;
    const parts = [];
    let lastIndex = 0;

    while ((match = regex.exec(template)) !== null) {
      if (match.index > lastIndex) {
        parts.push({ type: "static", value: template.slice(lastIndex, match.index) });
      }

      parts.push({ type: "token", flag: match[1], value: match[2] });
      lastIndex = regex.lastIndex;
    }

    if (lastIndex < template.length) {
      parts.push({ type: "static", value: template.slice(lastIndex) });
    }

    tokenCache.set(template, parts);
    return parts;
  }

  function renderTemplateString(template, data, syntax) {
    const parts = tokenize(template, syntax);
    let output = "";
    let index = 0;

    while (index < parts.length) {
      const part = parts[index];

      if (part.type === "static") {
        output += part.value;
        index++;
        continue;
      }

      if (part.flag === "/") {
        index++;
        continue;
      }

      let tokenValue;
      const innerParts = [];
      let cursor = index + 1;
      let foundClosing = false;

      while (cursor < parts.length) {
        const innerPart = parts[cursor];

        if (
          innerPart.type === "token" &&
          innerPart.flag === "/" &&
          innerPart.value === part.value
        ) {
          foundClosing = true;
          break;
        }

        innerParts.push(innerPart);
        cursor++;
      }

      try {
        tokenValue = applyFilter("token", part.value, data, template);
      } catch (error) {
        console.warn(error.message);
        tokenValue = "";
      }

      if (foundClosing) {
        const innerTemplate = innerParts
          .map((innerPart) =>
            innerPart.type === "static"
              ? innerPart.value
              : `${syntax.start}${innerPart.flag ? innerPart.flag : ""}${innerPart.value}${syntax.end}`
          )
          .join("");

        if (typeof tokenValue === "boolean") {
          output += tokenValue ? renderTemplateString(innerTemplate, data, syntax) : "";
        } else if (typeof tokenValue === "object") {
          for (const key in tokenValue) {
            if (tokenValue.hasOwnProperty(key)) {
              const localData = Object.assign({}, tokenValue[key], {
                _key: key,
                _value: tokenValue[key]
              });

              let rendered = renderTemplateString(innerTemplate, localData, syntax).trim();
              const potionKey = "potion_" + potionIndex++;
              potionMap.set(potionKey, localData);

              rendered = rendered.replace(
                /^\s*<([a-zA-Z0-9-]+)/,
                `<$1 data-potion-key="${potionKey}"`
              );

              output += rendered;
            }
          }
        } else {
          output += tokenValue;
        }

        index = cursor + 1;
      } else {
        output += tokenValue;
        index++;
      }
    }

    return output;
  }

  const potionStore = potionMap;

  function bindEvents(element, data) {
    [...element.attributes]
      .filter((attr) => attr.name.startsWith("@"))
      .forEach((attr) => {
        const eventParts = attr.name.slice(1).split(".");
        const eventName = eventParts[0];
        const modifiers = eventParts.slice(1);
        const parsed = attr.value.match(/^(\w+)(?:\((.*)\))?$/);

        if (!parsed) {
          console.warn("Potion: impossible de parser l'expression de l'événement:", attr.value);
          return;
        }

        const fnName = parsed[1];
        const rawArgs = parsed[2] || "";

        const localContext = (function findLocalContext(node, fallback) {
          let current = node;
          while (current && current !== document.body) {
            const key = current.getAttribute("data-potion-key");
            if (key) {
              const value = potionStore.get(key);
              if (value !== undefined) return value;
            }
            current = current.parentElement;
          }
          return fallback;
        })(element, data);

        const args = rawArgs
          ? rawArgs.split(",").map((arg) => {
              const trimmed = arg.trim();
              if (trimmed === "true") return true;
              if (trimmed === "false") return false;
              if (!isNaN(trimmed)) return Number(trimmed);
              const strMatch = trimmed.match(/^["'](.*)["']$/);
              return strMatch ? strMatch[1] : localContext[trimmed] || trimmed;
            })
          : [];

        const fn =
          typeof localContext[fnName] === "function"
            ? localContext[fnName]
            : typeof data[fnName] === "function"
            ? data[fnName]
            : null;

        if (typeof fn === "function") {
          element.removeEventListener(eventName, element._boundEvents?.[eventName]);

          const handler = (event) => {
            if (modifiers.includes("self") && event.target !== event.currentTarget) return;
            if (modifiers.includes("prevent")) event.preventDefault();
            if (modifiers.includes("stop")) event.stopPropagation();
            if (modifiers.includes("stopImmediate") && event.stopImmediatePropagation) {
              event.stopImmediatePropagation();
            }

            const context = { ...data, ...localContext };
            fn.call(context, event, ...args);
          };

          element._boundEvents = {
            ...element._boundEvents,
            [eventName]: handler
          };

          const options = {};
          if (modifiers.includes("capture")) options.capture = true;
          if (modifiers.includes("once")) options.once = true;
          if (modifiers.includes("passive")) options.passive = true;

          element.addEventListener(eventName, handler, options);
        } else {
          console.warn(`Potion: function '${fnName}' not found in local context or data.`);
        }

        element.removeAttribute(attr.name);
      });
  }

  function morphDom(currentNode, nextNode) {
    if (currentNode.nodeType === nextNode.nodeType && currentNode.nodeName === nextNode.nodeName) {
      if (currentNode.nodeType !== Node.TEXT_NODE) {
        if (currentNode.nodeType === Node.ELEMENT_NODE) {
          Array.from(nextNode.attributes).forEach((attr) => {
            if (
              !attr.name.startsWith("@") &&
              !attr.name.startsWith("#") &&
              currentNode.getAttribute(attr.name) !== attr.value
            ) {
              currentNode.setAttribute(attr.name, attr.value);
            }
          });

          Array.from(currentNode.attributes).forEach((attr) => {
            if (
              !attr.name.startsWith("@") &&
              !attr.name.startsWith("#") &&
              !nextNode.hasAttribute(attr.name)
            ) {
              currentNode.removeAttribute(attr.name);
            }
          });

          const currentChildren = Array.from(currentNode.childNodes);
          const nextChildren = Array.from(nextNode.childNodes);
          const max = Math.max(currentChildren.length, nextChildren.length);

          for (let i = 0; i < max; i++) {
            if (i >= currentChildren.length) {
              currentNode.appendChild(nextChildren[i].cloneNode(true));
            } else if (i >= nextChildren.length) {
              currentNode.removeChild(currentChildren[i]);
            } else {
              morphDom(currentChildren[i], nextChildren[i]);
            }
          }
        }
      } else if (currentNode.textContent !== nextNode.textContent) {
        currentNode.textContent = nextNode.textContent;
      }
    } else {
      currentNode.parentNode.replaceChild(nextNode.cloneNode(true), currentNode);
    }
  }

  function patchRoot(root, html) {
    const tagName = root.tagName.toLowerCase();
    const next = new DOMParser()
      .parseFromString(`<${tagName}>${html}</${tagName}>`, "text/html")
      .body.firstChild;

    [...root.attributes].forEach((attr) => {
      next.setAttribute(attr.name, attr.value);
    });

    morphDom(root, next);
  }

  const proxyCache = new WeakMap();

  function makeReactive(obj, onChange, maxDepth = Infinity, depth = 0) {
    if (typeof obj !== "object" || obj === null) return obj;
    if (depth >= maxDepth) return obj;
    if (proxyCache.has(obj)) return proxyCache.get(obj);

    const proxy = new Proxy(obj, {
      get(target, prop) {
        return makeReactive(Reflect.get(target, prop), onChange, maxDepth, depth + 1);
      },
      set(target, prop, value) {
        const oldValue = target[prop];
        const result = Reflect.set(target, prop, value);
        if (oldValue !== value) onChange();
        return result;
      }
    });

    proxyCache.set(obj, proxy);
    return proxy;
  }

  let templateRegistry = {};
  let initTriggered = false;

  let syntax = {
    start: "[",
    end: "]",
    path: "[a-z0-9_$][\\.a-z0-9_]*",
    type: "template/potion",
    attr: "data-name",
    tag: "div",
    class: ""
  };

  function resolveTemplate(template, data) {
    data = withStore(data);

    if (!initTriggered) {
      initTriggered = true;
      applyFilter("init", template, data);
    }

    template = applyFilter("templateBefore", template, data);
    if (!template.includes(syntax.start)) template = templateRegistry[template] || template;
    template = applyFilter("template", template, data);

    if (template && data !== undefined) {
      template = renderTemplateString(template, data, syntax);
    }

    return applyFilter("templateAfter", template, data);
  }

  function mountTemplate(templateNode, data, options) {
    options = { ...syntax, ...options };
    data = withStore(data);

    const html = resolveTemplate(templateNode.innerHTML, data);
    let root;
    const requestedTag = options.tag;

    root =
      !requestedTag || document.createElement(requestedTag) instanceof HTMLUnknownElement
        ? document.createElement(syntax.tag)
        : document.createElement(requestedTag);

    root.innerHTML = html;

    [...templateNode.attributes].forEach((attr) => {
      if (attr.name !== "type") root.setAttribute(attr.name, attr.value);
    });

    if (options.class) {
      root.classList.add(...options.class.split(" "));
    }

    data.$root = root;

    (function collectRefs(node, ctx) {
      const refs = {};
      node.querySelectorAll("[\\#ref]").forEach((refNode) => {
        const refName = refNode.getAttribute("#ref");
        if (refName) {
          refs[refName] = refNode;
          refNode.removeAttribute("#ref");
        }
      });
      ctx.$refs = Object.assign({}, ctx.$refs, refs);
    })(root, data);

    bindEvents(root, data);
    root.querySelectorAll("*").forEach((node) => bindEvents(node, data));

    templateNode.parentNode.replaceChild(root, templateNode);
    return root;
  }

  function render(template, data) {
    return resolveTemplate(template, data);
  }

  if (typeof window !== "undefined") {
    document.querySelectorAll(`template[type="${syntax.type}"]`).forEach((templateNode) => {
      const name = templateNode.getAttribute(syntax.attr);
      templateRegistry[name] = templateNode.innerHTML;
    });
  }

  render.sync = function (name, data, options) {
    const templateNode = document.querySelector(`template[data-name='${name}']`);
    if (!templateNode) throw new Error(`Potion: template with name '${name}' not found`);

    data = withStore(data);
    const originalHtml = templateNode.innerHTML;

    let rerender = () => {};
    const reactiveData = makeReactive(data, () => rerender());
    const root = mountTemplate(templateNode, reactiveData, options);

    rerender = () => {
      const html = resolveTemplate(originalHtml, reactiveData);
      patchRoot(root, html);
      bindEvents(root, reactiveData);
      root.querySelectorAll("*").forEach((node) => bindEvents(node, reactiveData));
    };

    return reactiveData;
  };

  render.render = function (name, data, options) {
    const templateNode = document.querySelector(`template[data-name='${name}']`);
    if (!templateNode) throw new Error(`Potion: template with name '${name}' not found`);
    return mountTemplate(templateNode, data, options);
  };

  render.addFilter = addFilter;
  render.applyFilter = applyFilter;

  const instance = {
    isLogged: _userdata.session_logged_in,
    store: [],
    unread: null,
    ..._userdata.session_logged_in && {
      syncStore: render.sync("all_notifs", {
        notifs: [],
        isEmpty: true,
        text: "Aucune notification"
      })
    },
    ..._userdata.session_logged_in && {
      syncUnread: render.sync("unread_notifs", { count: "" })
    },
    refresh: 0,
    users: {},
    disableIcon: false,
    type: {
      0: { name: "private_msg", icon: '<i class="bi bi-envelope-fill"></i>' },
      1: { name: "notif_report", icon: '<i class="bi bi-flag-fill"></i>' },
      2: { name: "friend_request", icon: '<i class="bi bi-person-fill-add"></i>' },
      3: { name: "group_req", icon: '<i class="bi bi-people-fill"></i>' },
      4: { name: "friend_conv", icon: '<i class="bi bi-people-fill"></i>' },
      5: { name: "wall_msg", icon: '<i class="bi bi-chat-fill"></i>' },
      6: { name: "abuse", icon: '<i class="bi bi-flag-fill"></i>' },
      7: { name: "topic_watch", icon: '<i class="bi bi-chat-fill"></i>' },
      8: { name: "mention", icon: '<i class="bi bi-at"></i>' },
      9: { name: "hashtag", icon: '<i class="bi bi-hash"></i>' },
      10: { name: "advert", icon: '<i class="bi bi-flag-fill"></i>' },
      11: { name: "like", icon: '<i class="bi bi-heart-fill"></i>' },
      12: { name: "dislike", icon: '<i class="bi bi-heart-half"></i>' },
      13: { name: "forum_watch", icon: '<i class="bi bi-chat-left-fill"></i>' },
      14: { name: "new_award", icon: '<i class="bi bi-star-fill"></i>' },
      15: { name: "follower_new_topic", icon: '<i class="bi bi-chat-left-fill"></i>' },
      16: { name: "follower_new_post", icon: '<i class="bi bi-chat-fill"></i>' }
    },

    init: async function (options = {}) {
      if (!_userdata.session_logged_in) return;

      bindPanelToggle({
        button: options.button || "#notiffi_button",
        panel: options.panel || "#notiffi_panel"
      });

      if (options.disableIcon == 1) {
        this.disableIcon = true;
      }

      if (options.icons) {
        for (const key in options.icons) {
          if (this.type[key]) this.type[key].icon = options.icons[key];
        }
      }

      this.manageNotifications();

      Toolbar = this.interceptMethodCalls(Toolbar, async (methodName) => {
        if (methodName === "refresh") {
          this.refresh++;

          const data = await (async function () {
            try {
              const response = await fetch("/notif");
              if (!response.ok) {
                throw new Error(`Failed to fetch: ${response.status} ${response.statusText}`);
              }
              return await response.json();
            } catch (error) {
              console.error("Error fetching notifications:", error);
            }
          })();

          this.store = data.store;
          this.displayNotifications(this.store);
          this.unread = data.unread;
          this.handleUnread(this.unread);

          if (
            this.refresh > 1 &&
            this.store.length &&
            !document.querySelector(`[data-notif-id="${this.store.at(-1).text.id}"]`)
          ) {
            this.alertNotif(options.timeout ? options.timeout : 5000, this.store.at(-1));
          }
        }
      });
    },

    alertNotif: async function (timeout, notification) {
      const { from, type } = notification.text;
      let avatar = "";

      if (from && from.name !== "Anonymous") {
        const userData = await getUserAvatar(from);
        avatar = userData ? userData.avatar : "";
      }

      const html = render("alert_notif", {
        alert: {
          type: this.type[type].name,
          icon: this.type[type].icon,
          avatar: type === 14 ? renderAward(notification) : avatar,
          text: Toolbar.compileNotif(notification),
          time: notification.time
        }
      });

      const alertNode = new DOMParser().parseFromString(html, "text/html").body.firstChild;
      document.body.appendChild(alertNode);

      alertNode.getBoundingClientRect();

      requestAnimationFrame(() => {
        alertNode.classList.add("up");
      });

      setTimeout(() => {
        alertNode.classList.remove("up");
        setTimeout(() => alertNode.remove(), 1000);
      }, timeout);

      document.body.addEventListener("click", (event) => {
        if (event.target.closest("#alert_dismiss")) {
          alertNode.classList.remove("up");
          setTimeout(() => alertNode.remove(), 1000);
        }
      });
    },

    renderNotif: async function (notifications) {
      const rendered = [];

      for (const notification of notifications) {
        const { id, from, type } = notification.text;
        let avatar = "";

        if (from && from.name !== "Anonymous") {
          const userData = await getUserAvatar(from);
          avatar = userData ? userData.avatar : "";
        }

        rendered.push({
          id,
          read: notification.read ? "" : "unread",
          type: this.type[type].name,
          ...!this.disableIcon && { icon: this.type[type].icon },
          avatar: type === 14 ? renderAward(notification) : avatar,
          text: Toolbar.compileNotif(notification),
          time: notification.time,
          async deleteNotif(event) {
            const notifId = event.target.closest("[data-notif-id]").dataset.notifId;
            const store = await deleteNotification(notifId, this.channel);
            instance.store = store;
            instance.displayNotifications();
            instance.handleUnread();
          }
        });
      }

      return rendered.reverse();
    },

    displayNotifications: async function () {
      if (this.store.length === 0) {
        this.syncStore.notifs = [];
        this.syncStore.isEmpty = true;
      } else {
        this.syncStore.notifs = await this.renderNotif(this.store);
        this.syncStore.isEmpty = false;
      }
    },

    manageNotifications: function () {
      const buttons = {
        deleteAll: document.querySelector("#notiffi_delete_all"),
        markAllRead: document.querySelector("#notiffi_mark_as_read")
      };

      const actions = {
        deleteAll: async () => {
          const ids = this.store.map((item) => item.text.id);
          if (await deleteAllNotifications(ids)) {
            while (this.store.length > 0) this.store.pop();
            this.displayNotifications();
            this.handleUnread();
          }
        },
        markAllRead: async () => {
          const ids = this.store.filter((item) => !item.read).map((item) => item.text.id);
          if (await markNotificationsAsRead(ids)) {
            this.store.forEach((item) => {
              item.read = true;
            });
            this.displayNotifications();
            this.handleUnread();
          }
        }
      };

      for (const key in buttons) {
        const button = buttons[key];
        if (!button) {
          console.error(`NOTIFFI: Le bouton ${key} est introuvable.`);
          return;
        }
        button.addEventListener("click", actions[key]);
      }
    },

    handleUnread: function () {
      const unreadCount = this.store.filter((item) => !item.read).length;
      this.unread = unreadCount;
      this.syncUnread.count = unreadCount ? this.unread : "";
    },

    interceptMethodCalls: function (target, callback) {
      return new Proxy(target, {
        get(targetObj, prop) {
          return typeof targetObj[prop] === "function"
            ? new Proxy(targetObj[prop], {
                apply(fn, thisArg, args) {
                  callback(prop, args);
                  return Reflect.apply(fn, thisArg, args);
                }
              })
            : Reflect.get(targetObj, prop);
        }
      });
    }
  };

  return instance;
})();
