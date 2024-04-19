(function (window) {
	ORA.recommender = {};
	var scriptExec = null;
	var cookieManager=null;
	var storage=null;
	var loader =null;
	var historyStoreMode=null;
	var recoTagMajorVersion="4";
	var lastViewedProduct=null;
	var productIDsList=null;

	ORA.Event.RECO_PRODUCT_READY = 'recommender_product_ready';

	function mergeParams(obj1, obj2) {
	if(!isNullOrUndefined(obj1) && !isObject(obj1) && isNullOrUndefined(obj2)) {
		return obj1;
	}

	obj1 = isNullOrUndefined(obj1) ? {} : obj1;
	obj2 = isNullOrUndefined(obj2) ? {} : obj2;

	if (!isObject(obj2)) {
		return obj2;
	}
	var res = {};
	if (isObject(obj1)) {
		foreach(obj1, function (value, key) {
			res[key] = value;
		});
	}
	foreach(obj2, function (value, key) {
		if (!res[key]) {
			res[key] = value;
		} else if (isArray(res[key]) && isArray(obj2[key])) {
			res[key] = res[key].concat(value);
		} else {
			res[key] = mergeParams(res[key], value);
		}
	});
	return res;
}


function isArray(a) {
	return Object.prototype.toString.call(a) === '[object Array]';
}

function isString(s) {
	return typeIs(s, 'string');
}

function isUndefined(u) {
	return typeIs(u, 'undefined');
}

function isNullOrUndefined(u) {
	return isUndefined(u) || u === null;
}

function typeIs(obj, type) {
	return typeof obj === type;
}

function isObject(o) {
	return typeIs(o, 'object');
}

function hasProperty(obj, name) {
	return Object.prototype.hasOwnProperty.call(obj, name);
}

function foreach(obj, processor) {
	for (var field in obj) {
		if (hasProperty(obj, field)) {
			processor(obj[field], field);
		}
	}
}
function  getCookieDomainFromHostUrl(){
	return  (location.hostname.match(/^[\d.]+$|/)[0] || ("." + (location.hostname.match(/[^.]+\.(\w{2,3}\.\w{2}|\w{2,})$/) || [location.hostname])[0])).replace(/^www\./i, '');
}

function objToUri(obj) {
	if (!isObject(obj)) {
		return obj;
	}
	if (isArray(obj)) {
		return obj.join(';');
	}
	var str = [];
	foreach(obj, function (value, key) {
		if (isArray(value)) {
			foreach(value, function (subValue) {
				str.push(key + '=' + subValue);
			});
		} else {
			str.push(key + '=' + encodeURIComponent(value));
		}
	});
	return str.join(';');
}

function hasStringProperty(obj, name) {
	return hasProperty(obj, name) && isString(obj[name]);
}

function parseValueViaJson(value) {
	var result = JSON.parse('{"v":' + value + '}');
	return ('v' in result) ? result['v'] : result;
}

function renderValueViaJson(value) {
	value = JSON.stringify({v: value});
	return value.substring(5, value.length - 1);
}

function CookieManager(conf, window) {
	var self = this;
	var encode = encodeURIComponent;
	var decode = decodeURIComponent;

	function parseCookies() {
		var c = window.document.cookie.split(/;\s+/g);
		var cookies = {};
		for (var i = 0; i < c.length; i++) {
			var p = c[i].split(/[;=]/);
			if (p.length > 1) {
				cookies[p[0]] = p[1];
			}
		}
		return cookies;
	}

	function futureDate(days) {
		var d = new window.Date();
		d.setTime(d.getTime() + days * 864e5);
		return d;
	}

	self.set = function (name, value, days, raw) {
		if (!raw) {
			value = encode(value);
		}
		var cookieStorageDomain = conf.storage.domain || getCookieDomainFromHostUrl();
		window.document.cookie = encode(name) + '=' + value +
			';domain=' + cookieStorageDomain + ';path=/' +
			(days ? ';expires=' + futureDate(days).toGMTString() : '') +
			(conf.storage.secure ? ';secure;sameSite=none' : '');
		return self;
	};

	self.remove = function (name) {
		self.set(name, '', -1);
		return self;
	};

	self.removeAll = function (pattern) {
		if (pattern) {
			var cookies = parseCookies();
			foreach(cookies, function (value, key) {
				if (new RegExp("^" + pattern).test(key)) {
					self.remove(decode(key));
				}
			});
		}
	};

	self.get = function (name, raw) {
		var r = new RegExp('(?:^|; )' + encode(name).replace(/([.$?*|{}()[\]\\/+^])/g, '\\$1') + 			'=([^;]+)'),
			match = window.document.cookie.match(r);
		var result;
		if (match) {
			var value = match[1];
			result = raw ? value : decode(value);
		}
		return result;
	};

	self.getAll = function (pattern, raw) {
		var result;
		if (pattern) {
			var cookies = parseCookies();
			result = {};
			foreach(cookies, function (value, key) {
				if (new RegExp("^" + pattern).test(key)) {
					result[decode(key)] = (raw ? value : decode(value));
				}
			});
		}
		return result;
	};

	return this;
}

/**
 * @class
 * @classdesc Designed to store the data inside the browser as separated cookies.
 * @constructor
 * @param {CookieManager} cookieMan wrapper over cookie storage
 * @param {Settings} settings mmapi settings obj.
 * @param {string} [namespace=def] namespace of storage
 * @param {Window} window global object
 */
function CookieStorage(cookieMan, conf, namespace, window) {
	namespace = (namespace === 'mmengine' ? 'e' : null) || namespace || 'def';

	function getPrefix() {
		return conf.storage.prefix + '.' + namespace + '.';
	}

	function getPrefixRx(prefix) {
		return prefix.replace(/\./g, '\\.');
	}

	/**
	 * Returns all or specified data stored in storage.
	 * @param {string} [name] Name of the variable. Should be undefined if you need to receive all data for namespace.
	 * @returns {*} Return specified data or object(key/value) with all data for namespace
	 */
	this.get = function (name) {
		var prefix = getPrefix();
		if (!name) {
			var prefixRx = getPrefixRx(prefix);
			var cookies = cookieMan.getAll(prefixRx);
			var s = prefix.length;
			var result = {};
			foreach(cookies, function (value, key) {
				result[key.substring(s)] = parseValueViaJson(value);
			});
			return result;
		}

		var value = cookieMan.get(prefix + name);
		if (value) {
			return parseValueViaJson(value);
		}
		return value;
	};

	/**
	 * Sets up a variable in storage.
	 * @param {string} name Name of the variable to store
	 * @param {*} value Value of the variable to store
	 * @param {number} [expires=0] Number of days in which the variable expires and should be removed from storage. 0 means session lifetime.
	 * @returns {CookieStorage}
	 */
	this.set = function (name, value, expires) {
		var prefix = getPrefix();
		if (value === null || isUndefined(value)) {
			cookieMan.remove(prefix + name);
		} else {
			value = renderValueViaJson(value);
			// use the lowest value - user-specified or out-of-the-box
			var expiration = isUndefined(expires) ? 0 : Math.min(conf.storage.expiration, expires);
			cookieMan.set(prefix + name, value, expiration);
		}
		return this;
	};

	/**
	 * Clears all saved data in current storage
	 * @returns {CookieStorage}
	 */
	this.removeAll = function () {
		cookieMan.removeAll(getPrefixRx(getPrefix()));
		return this;
	};
}

/**
 * @class
 * @classDesc Loader initiates first request to CG
 * responsible for loading of packages and passing CG response to it
 * provides ability to scripts to request CG
 * provides settings
 * @constructor
 * @param {Settings} settings all settings container
 * @param {Storage} storage factory of storage
 * @param {Window} window global object
 */
function Loader(conf, storage, window) {
    var clContainerName = 'json_response';
    this.ratingTrackingPage = 'ratingEvent';
    var requestId = 1;
    var document = window.document;
    var validationFunction;
    var responseId = 0;
    var self = this;
    self.mergeParams=mergeParams;
    this.conf=conf;
    var storages = [];
    var storeStrategies = {persistent: 0, deferredRequest: 1, request: 2, page: 3};
    var enable=true;

    function getParamsFromStorage() {
        return self.mergeParams(storages[storeStrategies.persistent].get(), self.mergeParams(storages[storeStrategies.deferredRequest].get(),
            self.mergeParams(storages[storeStrategies.page].get(), storages[storeStrategies.request].get())));
    }

    function disposeScriptTag(scriptId) {
        var scrTag = document.getElementById(scriptId);
        if (scrTag && scrTag.parentNode) {
            scrTag.parentNode.removeChild(scrTag);
        } else {
            if (scrTag) {
                scrTag.removeAttribute('src');
            }
        }
    }

    function storePersistData(result) {
        var persistentData = result && result.PersistData || [];
        var updateId = result && result.SystemData && result.SystemData[0] && result.SystemData[0].ResponseId || 0;
        if (updateId >= responseId) {
            for (var i = persistentData.length - 1; i >= 0; i--) {
                self.setParam(persistentData[i].Name, persistentData[i].Value, storeStrategies.persistent, persistentData[i].Expiration);
            }
            responseId = updateId;
        }
    }


    function sendPosttoDAPI(diApiUrl,newActionsPayload,visitorID){
        var xhr = new XMLHttpRequest();
        xhr.open("POST", diApiUrl);
        xhr.setRequestHeader("Accept", "application/json");
        xhr.setRequestHeader("Content-Type", "application/json");

        var data = {
            "visitor_id":visitorID,
            "actions" : newActionsPayload
        };

        xhr.send(JSON.stringify(data));
    }

    function processResponse(requestId, response, callback) {
        storePersistData(response);
        callback(!!response); //call callback and say result of execution(success/error)
    }

     this.request=function(rUrl, callback) {
        var scriptId = 'maxreco.' + requestId;
        (function (reqId, callback) {
            window[clContainerName][reqId] = function (result) {
                disposeScriptTag(scriptId);
                var runResponseCalled = false;
                var runResponse = function () {
                    if (!runResponseCalled) {
                        runResponseCalled = true;
                        processResponse(reqId, result, callback);
                    }
                };
                if (!isUndefined(validationFunction)) {
                    validationFunction(result, runResponse);
                    if (!runResponseCalled) {
                        setAsync(true);
                    }
                } else {
                    runResponse();
                }
                delete window[clContainerName][reqId];
            };
        })(requestId, callback);
        var attributes =   {id:scriptId,  src:rUrl};
        var head = document.getElementsByTagName('head')[0];
        var tagNode = document.createElement('script');
        foreach(attributes, function (value, key) {
            tagNode.setAttribute(key, value);
        });
        head.insertBefore(tagNode, head.lastChild);
        requestId++;
    }

    function buildSrvBaseUrl(srv, parameters) {
        if (hasStringProperty(parameters, 'un')) {
            srv = srv.indexOf("http:") === 0 ? srv.substring(5) : srv;
            srv = srv.indexOf("//") === 0 ? "https:" + srv : srv;
        }
        return srv;
    }

    this.clearStorage = function(){
        storages[storeStrategies.deferredRequest].removeAll();
        storages[storeStrategies.request].removeAll();
    }

    this.DAPICall = function(visitorID){
        try{
            var newActionsPayload =[];
            var storageParameters = getParamsFromStorage();

            // storageParameters["uv"]  has format like {"view":["1,1"]}  or {"purchase":["4,1","5,2","6,3"]}
            foreach(storageParameters["uv"], function (value, key) {
                for (var eventCount = 0; eventCount < value.length; eventCount++) {
                    var countProd= value[eventCount].split(",");
                    var action = {};
                    action['type'] = key;
                    action['product_id'] = countProd[1];
                    action['count'] = countProd[0]
                    newActionsPayload.push(action);
                }
            })
            var diapiUrl = this.conf.request.diapiServer +  "projects/" + this.conf.recoWorkSpaceId + "/actions";
            sendPosttoDAPI(diapiUrl,newActionsPayload,visitorID);
            this.clearStorage();
        }
        catch(e){
            ORA.Debug.error(e);
        }
    }

    this.CGCall = function (callback,params) {
        callback = callback || function () {};
        params= params || {};
        window[clContainerName] = window[clContainerName] || {};
        var loaderParameters = self.mergeParams(this.dynamicParams(),getParamsFromStorage());
        var allParameters = self.mergeParams(loaderParameters, params);
        var urlParameters = [];

        try{
            foreach(allParameters, function (value, key) {
                urlParameters.push(encodeURIComponent(key) + '=' + encodeURIComponent(objToUri(value)));
            });
            this.clearStorage();
            var srvBaseUrl = buildSrvBaseUrl(this.conf.request.server, allParameters);
            this.request(srvBaseUrl + urlParameters.join('&'), callback);
        }
        catch(e){
            ORA.Debug.error(e);
        }
    }

    this.baseStorage = storage.createBuilder();
    this.baseStorage.storeStrategy = storeStrategies;

    this.dynamicParams = function() {
        var clName = clContainerName + '[' + requestId + ']';
        var params = {};
        var screen = window.screen;
        params.dmn = this.conf.site;
        params.cok = self.baseStorage.testStorage();
        params.scrh = screen.height;
        params.scrw = screen.width;
        params.lver = this.conf.version || "2.0";
        params.jsncl = clName;
        params.ri = requestId;
        params.lto = -new Date().getTimezoneOffset();
        params.jrt = "s";
        return params;
    }

    this.setParam = function (name, value, storeStrategy, expire) {
        storages[storeStrategy].set(name, value, isUndefined(expire) ? this.conf.storage.expiration : expire);
        return this;
    };

    this.getParam = function (name, storeStrategy) {
        return storages[storeStrategy].get(name);
    };

    this.removeParam = function (name, storeStrategy) {
        storages[storeStrategy].set(name, '', -1);
        return this;
    };

    this.calcCookieDomain = calcCookieDomain(this.conf.storage.domain);//required in HL to inherit settings
    function calcCookieDomain(cookieDomain) {
        return cookieDomain || getCookieDomainFromHostUrl();
    }

    this.enable=function (){
        enable= true;
    }

    this.isEnabled=function(){
        return enable;
    }

    this.disable=function (){
        enable= false;
    }

    function memoryStorage() {
        var data = {};
        return {
            get: function (name) {
                return name ? data[name] : data;
            },
            set: function (name, value, expire) {
                if (parseInt(expire) < 0) {
                    delete data[name];
                } else {
                    data[name] = value;
                }
            },
            removeAll: function () {
                data = {};
            }
        };
    }

    this.setLocalHistory = function(eventT,product,storeName){
        historyStoreMode  = conf.historyStoreMode;
        if(historyStoreMode !== "Cookie" && historyStoreMode !== "Local Storage" ){
            ORA.Debug.debug("History Store mode Not Supported "+historyStoreMode);
            return;
        }
        var cookieHistoryStore = new CookieManager(conf, window);
        var isCandidateForSS= "Y";
        var viewEvent="V",purchaseEvent="P";
        var maxSupportedItems=3;
        var productEventMap={};
        var eventMapToStore = {};
        var cookieStoreName="";
        var productEncoded= window.btoa(product);
        var cookieLimitSize = 1700;
        var productlist=null;
        var recoTagVersionStored=null;


        try{
            var item=null;
            var eventValue=null;

            if(eventT && eventT.toLowerCase().includes("view")){
                eventValue=viewEvent;
            }else if(eventT && eventT.toLowerCase().includes("purchase")){
                eventValue=purchaseEvent;
            }else{
                return;
            }

            var currentTime= Math.floor(Date.now()/1000);

            if(historyStoreMode === "Local Storage"){
                item = (JSON.parse(localStorage.getItem(storeName)));
                recoTagVersionStored= (localStorage.getItem("recoTagMajorV"));
                if(!recoTagVersionStored){
                    localStorage.removeItem("lastEventTime");
                }
            }else if(historyStoreMode === "Cookie"){
                cookieStoreName=conf.storage.prefix+"."+storeName;
                var itemInCookie=  (cookieHistoryStore.get(cookieStoreName,true));
                if(itemInCookie){
                    item=JSON.parse(itemInCookie)
                }
                recoTagVersionStored= (cookieHistoryStore.get(conf.storage.prefix+".recoTagMajorV",true));
                if(!recoTagVersionStored){
                    cookieManager.set(conf.storage.prefix+".lastEventTime",'',-1,true)
                }
            }

            if(item){
                item=this.convertToLatestFormat(item,recoTagVersionStored);
                eventMapToStore = {};
                eventMapToStore[eventValue]= {D:currentTime};
                productEventMap[productEncoded] = eventMapToStore;
                item.push(productEventMap)

                var itemLength = (JSON.stringify(item)).length;
                while(itemLength > cookieLimitSize) {
                    item.splice(0, 1);
                    itemLength = (JSON.stringify(item)).length;
                }
                productlist = item;
            }else{
                eventMapToStore = {};
                productlist = [];
                eventMapToStore[eventValue]= {D:currentTime};
                productEventMap[productEncoded] = eventMapToStore;
                productlist.push(productEventMap);
            }
            if(historyStoreMode === "Local Storage"){
                localStorage.setItem(storeName,JSON.stringify(productlist));
                localStorage.setItem("recoTagMajorV",recoTagMajorVersion);
            }else if(historyStoreMode === "Cookie"){
                cookieHistoryStore.set(cookieStoreName,JSON.stringify(productlist),conf.storage.expiration,true);
                cookieHistoryStore.set(conf.storage.prefix+".recoTagMajorV",recoTagMajorVersion,conf.storage.expiration,true);
            }
            this.sendHistory(productlist);
        }catch(e){
           ORA.Debug.debug(e)
        }
    }

    this.sendHistory= function(historyItemList){
        var finalValue="";
        if(historyItemList)
        {
            for (var i=0;i<historyItemList.length;i++) {
                var historyItem = historyItemList[i];
                for (var productIDKey in historyItem) {
                    if (historyItem.hasOwnProperty(productIDKey)) {
                        for (var eventKey in historyItem[productIDKey]) {
                            if (historyItem[productIDKey].hasOwnProperty(eventKey)) {
                                if (finalValue.length > 0) {
                                    finalValue += ";";
                                }
                                finalValue = finalValue + (productIDKey) + ":" + eventKey + ":" + historyItem[productIDKey][eventKey].D;
                            }
                        }
                    }
                }
            }

            var historyDataFull= finalValue.split(";");
            var historyDataFirst="";
            var historyDataSecond="";
            var historyDataThird="";

            var rsysItemLimit = (1500/3)-10;

            for(var index=0;index<historyDataFull.length;index++){
                var dataLength= historyDataFull[index].length;
                if((historyDataFirst.length+dataLength) <= rsysItemLimit){
                    if(historyDataFirst.length>0){
                        historyDataFirst=historyDataFirst+";";
                    }
                    historyDataFirst= historyDataFirst+historyDataFull[index];
                }else if((historyDataSecond.length+dataLength) <= rsysItemLimit){
                    if(historyDataSecond.length>0){
                        historyDataSecond=historyDataSecond+";";
                    }
                    historyDataSecond= historyDataSecond+historyDataFull[index];
                }else if((historyDataThird.length+dataLength) <= rsysItemLimit){
                    if(historyDataThird.length>0){
                        historyDataThird=historyDataThird+";";
                    }
                    historyDataThird= historyDataThird+historyDataFull[index];
                }else{
                    break;
                }
            }
            ORA.click({data: {"wt.tx_rechist1":(historyDataFirst),"wt.tx_rechist2":(historyDataSecond),"wt.tx_rechist3":(historyDataThird)}})

        }
    }

    this.convertToLatestFormat=function(item,recoTagV){
        if(!recoTagV){
            var productResultMap = new Map();
            for (var productIDKey in item) {
                if (item.hasOwnProperty(productIDKey)) {
                    for (var eventKey in item[productIDKey]) {
                        if (item[productIDKey].hasOwnProperty(eventKey)) {
                            var productEventSession = productIDKey + ":" + eventKey;
                            productResultMap.set(productEventSession, item[productIDKey][eventKey].D);
                        }
                    }
                }
            }

            if (productResultMap && productResultMap.size > 0) {
                var sortedProdEventArray =  Array.from(productResultMap)
                    .sort(function (a, b) {
                        return a[1] - b[1]; });

                var productlist = [];
                for (var nOfProd = 0; nOfProd <  sortedProdEventArray.length; nOfProd++) {
                    var prodEventArray =  sortedProdEventArray[nOfProd][0].split(":")
                    var productEventMap={};
                    var eventMapToStore = {};
                    eventMapToStore[prodEventArray[1]]= {D: sortedProdEventArray[nOfProd][1]};
                    productEventMap[prodEventArray[0]] = eventMapToStore;
                    productlist.push(productEventMap);
                }
                return productlist;
            }

        } else if(recoTagV !== recoTagMajorVersion){
            ORA.Debug.debug("Not in the latest version");
            //To do Infuture for latest version changes.
        }
        return item;
    }

    function init() {
        storages[storeStrategies.persistent] = self.baseStorage('p');
        storages[storeStrategies.deferredRequest] = self.baseStorage('d');
        storages[storeStrategies.request] = memoryStorage();
        storages[storeStrategies.page] = memoryStorage();
    }
    init();
    return this;
}
function Rating(conf,loader){
    this.conf=conf;
    var payload=null;
    var visitorID = null
    this.send = function (name, value, attribute) {
        if (arguments.length > 0 && name && isString(name)) {
            // if name is specified and it is correct (value and attribute also can be specified)
            this.setRating(name, value, attribute);
        } else if (arguments.length !== 0) {
            // if name is specified and it is incorrect (name is required argument)
            ORA.Debug.error('You can\'t use rating.send without setting rating name');
            return;
        }
        // if name is correct or no arguments is specified (ratings can be set previously)
        if (this.hasRatings() && loader.isEnabled()) {
            if(this.conf.request && this.conf.request.diapiServer){
                loader.DAPICall(visitorID);
            }else{
                loader.CGCall(undefined, {pageid : loader.ratingTrackingPage});
            }
        } else {
            loader.clearStorage();
        }

    };

    function rating(name, value, attr, storeType) {
        var uv = loader.getParam('uv', storeType) || {};
        name = trim(name);
        uv[name] = uv[name] || [];
        if (value === undefined) {
            value = 1;
        }
        else if (value === '' || isNaN(value)) {
            value = 0;
            attr = 'NaN';
        }
        if(!attr && payload )
        {
            if(payload.length >0){
                uv[name].push(value + (payload ? ',' + encodeURIComponent(payload) : ''));
            }
        }else{
            uv[name].push(value + (attr ? ',' + encodeURIComponent(attr) : ''));
        }
        loader.setParam('uv', uv, storeType);
    }
    function trim(str) {
        return str.replace(/^\s+|\s+$/gm, '');
    }

    /**
     * @ignore
     * @protected
     * @description Capture rating parameters to a buffer, so all page ratings can be sent as one package
     * @param {string} name Name of the rating
     * @param {string|number} [value=1] rating value
     * @param {string} [attr=''] Rating attribute
     * @since 1.0
     */
    this.setRating = function (name, value, productId) {
        if(!productId && payload)
        {
            rating(name, value,payload,loader.baseStorage.storeStrategy.request);
            loader.setLocalHistory(name,payload,"localHistory")
        }else{
            rating(name, value, productId,loader.baseStorage.storeStrategy.request);
            loader.setLocalHistory(name,productId,"localHistory")
        }
    };

    /**
     * @ignore
     * @protected
     * @description Check has rating.
     * @returns {boolean}
     * @since 1.0
     */
    this.hasRatings = function () {
        var uvRequest = loader.getParam('uv', loader.baseStorage.storeStrategy.request) || {};
        var uvDeferred = loader.getParam('uv', loader.baseStorage.storeStrategy.deferredRequest) || {};
        return ((JSON.stringify(uvRequest) !== '{}') || (JSON.stringify(uvDeferred) !== '{}'));
    };

    /**
     * @public
     * @description Capture rating parameters to a buffer, so all page ratings can be sent as one package
     * @param {string} name Name of the Rating
     * @param {string|number} [value=1] Rating value
     * @param {string} [attribute=''] Rating attribute
     * @returns {Rating}
     * @since 1.0
     */
    this.set = function (name, value, attribute) {
        if (!name || !isString(name)) {
            ORA.Debug.error('You can\'t use rating.set without setting rating name.');
            return this;
        }
        this.setRating(name, value, attribute);
        return this;
    };

    this.setPayload = function(productID) {
        payload=productID;
    };

    this.setVisitorID = function(visitorId) {
        visitorID = visitorId;
    }
}


function ScriptExecutor(config){
    this.config = config;
    this.scriptExecutor = function (isSPA,visitorID) {
        var resps = [];
        try{
            var scriptsToExecute={};
            if(this.config && this.config.Data && this.config.Data.scripts) {
                scriptsToExecute = this.config.Data.scripts;
            }

            for(var i=0;i<scriptsToExecute.length;i++) {
                    var scriptFunction = getFunctionContentStr(scriptsToExecute[i]);
                    if(scriptFunction!==null){
                        resps.push(getResponseFromScript(scriptFunction));
                     }
            }
			ratings.setVisitorID(visitorID)
			ratings.send();
        } catch (e) {
			ORA.Debug.error(e);
            throw e;
        }
        return resps;
    };

	this.afterInitExecutor = function (isSPA) {
		var resps = [];
		try{
			var scriptsToExecute={};
			if(this.config && this.config.Extensions && this.config.Extensions.afterInit) {
				scriptsToExecute = this.config.Extensions.afterInit;
			}
			scriptsToExecute();
		} catch (e) {
			ORA.Debug.error(e);
			throw e;
		}
		return resps;
	};

    function getFunctionContentStr(script){
        var content = script.content;
		var include=false;
		var exclude=false;
		var includeUrl= script.includeURLs;
		var excludeUrl= script.excludeURLs;
		if(includeUrl) {
			for (var i = 0; i < includeUrl.length; i++) {
				if (includeUrl[i]) {
					var locationURL = window.location.href;
					var regexURL= new RegExp(includeUrl[i].replace(/\*|\?/g, '.*').replace(/([$?|{}()[\]\\/+^])/g, '\\$1'),"g");
					var res = regexURL.test(locationURL);
					if (res) {
						include = true;
						break;
					}
				}
			}

			if (include && excludeUrl) {
				for (i = 0; i < excludeUrl.length; i++) {
					if (excludeUrl[i]) {
						locationURL = window.location.href;
						regexURL= new RegExp(excludeUrl[i].replace(/\*/g, '.*').replace(/([$?|{}()[\]\\/+^])/g, '\\$1'),"g");
						res = regexURL.test(locationURL);
						if (res) {
							exclude = true;
							break;
						}
					}
				}
			}

			if (include && !exclude) {
				return content;
			}
		}
		return null;
    }

    function getResponseFromScript(scriptFunction){
        return scriptFunction();
    }

    return this;

}
function SPASupport(conf){

    if(conf.spaTrackURL){
        window.addEventListener('popstate', popStateHandler, false);
    }
    var elemQuerySelector = conf.spaSelector ? conf.spaSelector:"body";
    var targetNode = document.querySelector(elemQuerySelector);
    if(!targetNode){
        ORA.Debug.error("[SPASupportl] Unsupported targetNode ");
        return;
    }
    var observerConfig = {
        attributes: true,
        attributeFilter: ['src', 'href'],
        childList: true,
        subtree: true,
        characterData: true,
        attributeOldValue: true,
        characterDataOldValue: true
    };


    var observer = new MutationObserver(function () {
        observer.disconnect();
        spaScriptExecuteCallback();
        observer.observe(targetNode, observerConfig);
    });

    observer.observe(targetNode, observerConfig);

    function spaScriptExecuteCallback(){
        try{
            scriptExec.scriptExecutor(true);
        }catch(e){
            ORA.Debug.error("[ORA.recommender.executeCGCall.spaScriptExecuteCallback] Script Execution Error. Check Script ");
        }
    }

    function popStateHandler(){
        try{
            scriptExec.scriptExecutor(true);
        }catch(e){
            ORA.Debug.error("[ORA.recommender.executeCGCall.popStateHandler] Script Execution Error. Check Script ");
            console.log("[SpaHashHandler] Script Execution Error. Check Script ");
        }
    }
}
/**
 * @class
 * @classDesc Creates storage with predefined settings
 * @constructor
 * @param {Settings} settings all settings container
 * @param {CookieManager} cookieMan cookie manager instance
 * @param {Window} window global object
 */
function Storage(conf, cookieMan, window) {

	this.createBuilder = function() {
		var builder = function (namespace) {
			return new CookieStorage(cookieMan, conf, namespace, window);
		};
		builder.isSecure = conf.storage.secure;//required as field by HL to inherit setting
		builder.testStorage = testStorage;
		return builder;
	};
	/**
	 * Check that storage can save and read data.
	 * @returns {number} 1 if all work fine, 0 if not
	 */
	function testStorage() {
		var rnd = (new Date().getTime()).toString().slice(-5);
		cookieMan.set(conf.storage.prefix + '.tst', rnd, 10);
		var testResult = (cookieMan.get(conf.storage.prefix + '.tst', true) === rnd ? 1 : 0);
		cookieMan.remove(conf.storage.prefix + '.tst');
		return testResult;
	}
}

	ORA.recommender.setup = function(){
		if(window.recoInit){
			return ;
		}
		try{
			var conf= ORA.recommenderModule.prototype.oraConfigObj;
			historyStoreMode = conf.historyStoreMode;
			ORA.Debug.debug('SetUp:  Setting Up Objects');
			cookieManager = new CookieManager(conf, window);
			storage = new Storage(conf, cookieManager, window);
			loader = new Loader(conf, storage, window);
			if(conf.Extensions && conf.Extensions.beforeInit){
				var beforeInitFunction = (conf.Extensions.beforeInit);
				beforeInitFunction(loader);
			}
			scriptExec = new ScriptExecutor(conf);
			window.ratings = new Rating(conf,loader);
			if(conf.isSPA && conf.Data.Mode==="scripts"){
				ORA.Debug.debug("[ORA.recommender.setup]Setting up Spa support");
				if(document.readyState === "complete"){
					ORA.Debug.debug("Document is Ready, Setting up for SPA events");
					SPASupport(conf);
				}else{
					ORA.Debug.debug("[ORA.recommender.setup]addEventListener to register for SPA");
					window.onload=function() {
						ORA.Debug.debug('Registered for SPA events');
						SPASupport(conf);
					};
				}
			}
			if(historyStoreMode) {
				var cookieStoreName=conf.storage.prefix+".localHistory";
				var item=null;
				var recoTagMajorV=null;
				if (historyStoreMode === "Cookie") {
					item = (JSON.parse(localStorage.getItem("localHistory")));
					recoTagMajorV = (localStorage.getItem("recoTagMajorV"));
					if (item) {
						ORA.Debug.debug("Moving from LS to Cookie");
						cookieManager.set(cookieStoreName,JSON.stringify(item),conf.storage.expiration,true)
						if(recoTagMajorV){
							cookieManager.set(conf.storage.prefix+".recoTagMajorV",recoTagMajorV,conf.storage.expiration,true);
							localStorage.removeItem("recoTagMajorV");
						}
						localStorage.removeItem("localHistory");
						localStorage.removeItem("lastEventTime");
					}
				} else if (historyStoreMode === "Local Storage") {
					var itemInCookie=  (cookieManager.get(cookieStoreName,true));
					if(itemInCookie){
						item=JSON.parse(itemInCookie)
					}

					recoTagMajorV= (cookieManager.get(conf.storage.prefix+".recoTagMajorV",true));

					if (item) {
						localStorage.setItem("localHistory", JSON.stringify(item));
						cookieManager.set(cookieStoreName,'',-1,true)
						cookieManager.set(conf.storage.prefix+".lastEventTime",'',-1,true)
						if(recoTagMajorV){
							localStorage.setItem("recoTagMajorV", (recoTagMajorV));
							cookieManager.set(conf.storage.prefix+".recoTagMajorV",'',-1,true)
						}
						ORA.Debug.debug("Moving from Cookie to LS");
					}
				}
			}
		}catch(e){
			ORA.Debug.error(e);
		}
		if (scriptExec && loader.isEnabled()) {
			try {
				scriptExec.afterInitExecutor(false);
			} catch (e) {
				ORA.Debug.error("[ORA.recommender.executeCGCall] Script Execution Error. Check Script ");
			}
		}
		window.recoInit=true;
		ORA.fireEvent(new ORA.Event(ORA.Event.RECO_PRODUCT_READY, ORA.Event.STATUS_SUCCESS));
	}


	ORA.recommender.getClientIDAndExecute = function  (visitorID){
		scriptExec.scriptExecutor(false,visitorID);
	}

	// visitorID and productID will only be available in case of Analytics based Reco module and hence the mode is automatically switched based on productID.
	// For analytics mode this function cannot be called if we dont have product ID.
	ORA.recommender.executeCGCall = function (conf,productID,eventName,productUnits,visitorID){
		try {
			ORA.Debug.debug('executeCGCall:  Executing CGCall');
			if (!loader) {
				ORA.Debug.debug('Loader is not initiliazed');
				return;
			}
			if (productID) {
				var prodIDList = productID.split(";");
				var prodUnitsList = null;
				if (productUnits) {
					prodUnitsList = productUnits.toString().split(";");
					if (prodIDList.length !== prodUnitsList.length) {
						ORA.Debug.error("ProductID might have non Supported Character");
						return;
					}
				}
				else if (prodIDList.length > 1) {
					ORA.Debug.error("ProductID might have non Supported Character");
					return;
				}

				if(eventName.toLowerCase().includes("view")){
					lastViewedProduct=prodIDList[0];
					ORA.Debug.debug(lastViewedProduct);
				}

				if(visitorID){
					ratings.setVisitorID(visitorID);
				}

				for (var prodIdIndex = 0; prodIdIndex < prodIDList.length; prodIdIndex++) {
					ratings.setPayload(prodIDList[prodIdIndex]);
					if (eventName) {
						ratings.set(eventName, productUnits ? prodUnitsList[prodIdIndex] : 1);
					}
				}

				if (loader.isEnabled()) {
					ORA.Debug.debug("[ORA.recommender.executeCGCall] Sending Ratings ");
					ratings.send();
				}
			} else {
				if (scriptExec && loader.isEnabled()) {
					try {
						if(conf.request && conf.request.diapiServer){
							ORA.common.clientID.getClientID(ORA.recommender.getClientIDAndExecute);
						}else{
							scriptExec.scriptExecutor(false);
						}
					} catch (e) {
						ORA.Debug.error("[ORA.recommender.executeCGCall] Script Execution Error. Check Script ");
					}
				}
			}
		}catch (e) {
			ORA.Debug.error("[ORA.recommender.executeCGCall] failed");
		}
	}

	ORA.recommender.getHistory = function() {
		var conf = ORA.recommenderModule.prototype.oraConfigObj;
		historyStoreMode=conf.historyStoreMode;
		var item = null;
		var recoTagVersion=null;
		var cookieStoreName = conf.storage.prefix + ".localHistory";
		if (historyStoreMode && historyStoreMode === "Local Storage") {
			item = (JSON.parse(localStorage.getItem("localHistory")));
			recoTagVersion = localStorage.getItem("recoTagMajorV");
		} else if (historyStoreMode && historyStoreMode === "Cookie") {
			var itemInCookie = (cookieManager.get(cookieStoreName, true));
			if (itemInCookie) {
				item = JSON.parse(itemInCookie)
			}
			recoTagVersion= (cookieManager.get(conf.storage.prefix + ".recoTagMajorV", true));
		}
		if(loader) {
			item = loader.convertToLatestFormat(item, recoTagVersion);
		}
		return item;
	}
	ORA.recommender.popLastViewedProduct = function(){
		var tempLastViewedProduct = lastViewedProduct;
		lastViewedProduct=null;
		return tempLastViewedProduct;
	}

	ORA.recommender.getLastViewedProduct = function(){
		return lastViewedProduct;
	}

	ORA.recommender.setLastViewedProduct = function(productID){
		lastViewedProduct = productID;
	}

	ORA.recommender.setProductIDs = function(productIDs){
		productIDsList="";
		if(!productIDs){
			return;
		}
		var combinedProd="";
		if(Array. isArray(productIDs)){
			var totalProdCount=0;
			for ( var productID=0; productID<productIDs.length;productID++) {
				var currentProdId=productIDs[productID];
				if(!currentProdId || currentProdId.length >50){
					continue;
				}
				if(totalProdCount >=10){
					break;
				}
				totalProdCount++;
				if(combinedProd!=="") {
					combinedProd += ";" + window.btoa(currentProdId);
				}
				else{
					combinedProd=window.btoa(currentProdId);
				}
			}
		}else{
			combinedProd=window.btoa(productIDs);
		}
		productIDsList=combinedProd;
		if(productIDsList) {
			ORA.click({data: {"wt.tx_recsku": (productIDsList)}})
		}
	}


	ORA.recommender.getProductIDs =function () {
		return new Promise(function (resolve, reject) {
			var numberOfRetries =0;
			(function waitForData(){
				if (productIDsList){
					var prodList= productIDsList.split(";");
					for( var identi=0; identi<prodList.length;identi++){
						prodList[identi]=window.atob(prodList[identi]);
					}
					return resolve(prodList);
				}
				numberOfRetries++;
				if(numberOfRetries >=10){
					return reject();
				}
				setTimeout(waitForData, 500);
			})();
		});
	}

	ORA.setExecuteState("recommender", "ready");

	ORA.recommenderModule.CookieManager=CookieManager;
	ORA.recommenderModule.Loader=Loader;
	ORA.recommenderModule.Storage=Storage;
	ORA.recommenderModule.CookieStorage=CookieStorage;
	ORA.recommenderModule.SPASupport=SPASupport;
	ORA.recommenderModule.Rating=Rating;
	ORA.recommenderModule.ScriptExecutor=ScriptExecutor;
	ORA.recommenderModule.Setup=ORA.recommender.setup;
	ORA.recommenderModule.ExecuteCGCall=ORA.recommender.executeCGCall;
	ORA.recommenderModule.GetHistory=ORA.recommender.getHistory;
	ORA.recommenderModule.GetLastViewedProduct=ORA.recommender.getLastViewedProduct;
	ORA.recommenderModule.PopLastViewedProduct=ORA.recommender.popLastViewedProduct;

})(window);
