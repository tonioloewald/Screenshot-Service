#!/Users/tonio/phantomjs/bin/phantomjs --ignore-ssl-errors=yes

/**
    Screen Shot Service
    ===================

    Requires
    --------
    phantomjs -- https://github.com/ariya/phantomjs/
    Looks like it needs 70-130MB of RAM to handle one request at a time

    Launch this from the command line then call it using parameters such as:
    * url -- the url to be captured (required) -- protocol prefix is optional
    * format -- jpeg (default), png, pdf
    * paper -- letter (default)
    * orientation -- portrait (default), landscape
    * width -- 1024 is default
    * height -- 768 is default (usually irrelevant)
    * hilite -- text (regular expression) to hilite
    * stamp -- fontsize of url and time stamp (default 10 | 0 suppresses stamp)

    TODO
    ----
    * caching would be trivial to implement if needed
    * Flash plugin support?! Possible with SlimeJS but it isn't really headless
      Nike.com and HBO.com do not render
*/

/*jshint 
    noarg:true, noempty:true, eqeqeq:true, laxbreak:true, undef:true, 
    browser:true, devel:true, node:true, maxerr:50 
*/
/*global phantom */

var server = require('webserver').create(),
    fs = require('fs'),
    verbose = true,
    port = 8002,
    screenshots = '/tmp/screenshots/',
    hiliteColor = 'rgba(255,255,128,0.8)',
    textColor = 'rgba(0,0,0,0.8)';

if( !fs.exists(screenshots)){
    vlog('creating directory', screenshots);
    fs.makeDirectory(screenshots);
}
if( !fs.isDirectory(screenshots) || !fs.isWritable(screenshots) || !fs.isReadable(screenshots) ){
    console.log('Cannot use screenshots directory', screenshots);
    phantom.exit();
}

console.log("Listening on port -- e.g. localhost:8002/?url=http://google.com", port);

function vlog(){
    if( verbose ){
        console.log.apply(console, arguments);
    }
}

function jsonResponse(response, data){
    if( !response ){
        return;
    }
    response.header = { 'type': 'application/json' };
    response.write(JSON.stringify(data));
    response.close();
}

function errorResponse(response, msg, status){
    if( !response ){
        return;
    }
    if(!status){ 
        status = '400'; 
    }
    if(!msg){
        msg = "Bad Request";
    }
    vlog(msg);
    response.statusCode = status;
    response.header = { 'type': 'text/plain' };
    response.write( 'Status: ' + status + '\n' );
    response.write( 'Error: ' + msg );
    response.close();
}

function fileResponse(response, fileName){
    if( !response ){
        return;
    }

    var mimeType = fileName.substr(-4) === '.PDF' 
                        ? 'application/PDF' 
                        : 'image/' + (fileName.split('.').pop()).toUpperCase();
            
    if( fs.exists( fileName ) ){
        response.setEncoding('binary');
        response.writeHead(200, {'type': mimeType });
        response.write( fs.read( fileName, 'b' ) );
        response.close();
        vlog('served', fileName, mimeType);
    } else {
        vlog( fileName, 'missing' ); 
        errorResponse( response, 'file not found', 404 );
    }
}

// Detects the BlueHost warning page and clicks through it.
function isBlueHost(){
    var link = document.querySelectorAll('center span a');
    if( 
        link.length === 1 && 
        link[0].textContent.match(/access the Internet/) 
    ){
        // dispatching click event from first principles
        var evt = document.createEvent("MouseEvent");
        evt.initMouseEvent( 
            "click", 
            true, // bubble
            true, // cancelable
            window, null, 
            0, 0, 0, 0, // coordinates
            false, false, false, false, // modifier keys
            0, // left mouse button
            null
        );
        link[0].dispatchEvent(evt);
        return true;
    }
    return false;
}

function hilite(re, node, hiliteColor, textColor, depth, maxdepth){
    depth = depth || 0;
    maxdepth = maxdepth || 50;
    var result = 0;

    if( re.test('') ){
        console.error('regular expression matches empty string');
        return;
    }

    try {
        node = node || document.body;
        if( node.nodeType === 3 ){
            var match = node.data.match(re);
            if(match){
                var span = document.createElement('span');
                span.style.backgroundColor = hiliteColor;
                span.style.boxShadow = '0 0 8px 0 ' + hiliteColor;
                span.style.color = textColor;
                var wordNode = node.splitText(match.index);
                wordNode.splitText(match[0].length);
                var wordClone = wordNode.cloneNode(true);
                span.appendChild(wordClone);
                wordNode.parentNode.replaceChild(span, wordNode);
                result = 1;
            }
        } else if( depth === maxdepth ){
            console.log( 'Reached max recursion depth!', node );
        } else if( (node.nodeType === 1 && node.childNodes) && !/(script|style)/i.test(node.tagName) ) {
            for( var i = 0; i < node.childNodes.length; i++ ){
                i += hilite(re, node.childNodes[i], hiliteColor, textColor, depth + 1, maxdepth);
            }
        }
    } catch(e){
        result = 1; // prevent infinite loops
    } finally {
        return result;
    }
}

function stamp(info, fontSize, hiliteColor, textColor){
    try {
        var e = document.createElement('div'),
            f = document.createDocumentFragment();

        f.textContent = info;
        e.appendChild(f);
        e.style.position = 'absolute';
        e.style.top = 0;
        e.style.right = 0;
        e.style.padding = '2px 4px';
        e.style.backgroundColor = hiliteColor;
        e.style.color = textColor;
        e.style.fontFamily = 'Consolas,"Ubuntu Mono Regular",monospace';
        e.style.border = '1px solid ' + textColor;
        e.style.zIndex = 2147483647; // in front dammit! maxint 32-bit
        e.style.fontSize = fontSize + 'px';
        document.body.appendChild(e);
    } finally{
        // just trying to avoid hanging the page
    }
}

function processArguments( url ){
    var args = url.split('?'),
        settings = {};

    if( args.length > 1 ){
        args = args[1].split('&');
        for( var idx in args ){
            var parts = args[idx].split('=');
            parts[1] = decodeURIComponent( parts[1] );
            settings[ parts[0] ] = parts.length > 1 ? parts[1] : true;
            vlog( parts[0], parts[1] );
        }
        settings.width = settings.width || 1024;
        settings.height = settings.height || 768;
        settings.paper = settings.paper || 'letter';
        settings.format = settings.format || 'JPEG';
        settings.format = settings.format.toUpperCase();
        settings.delay = settings.delay || 0;
        settings.hilite = settings.hilite || false;
        settings.stamp = settings.stamp === undefined ? 10 : parseInt(settings.stamp,10);
        if( settings.url.search(/https?:\/\//) !== 0 ){
            settings.url = 'http://' + settings.url;
        }
    } else {
        settings.fileName = screenshots + args[0].substr(1); // strip leading slash!
    }

    return settings;
}

function makeScreenshot( settings, response ){
    var tempFile = (new Date()).getTime() + '.' + settings.format.toLowerCase(),
        page = require('webpage').create();

    vlog('screenshot creation request', settings.url, settings.format);

    page.viewportSize = { width: settings.width, height: settings.height };
    page.paperSize = { format: settings.paper, orientation: 'portrait', border: '0.5in' };
    page.open( settings.url, function(status){
        if(status === 'success'){
            if( page.evaluate(isBlueHost) ){
                vlog("Bypassing BlueHost... This screenshot will suck!");
            }
            if( settings.stamp ){
                var stampInfo = settings.url + ' ' + (new Date()).toISOString();
                settings.delay += 0.1;
                vlog('stamping', stampInfo);
                page.evaluate(stamp, stampInfo, settings.stamp, hiliteColor, textColor);
            }
            if( settings.hilite ){
                var re = new RegExp(settings.hilite, 'i');
                if( re.test('') ){
                    vlog('cannot hilite -- bad search terms');
                } else {
                    settings.delay += 1; // allow a second for hiliting to execute
                    vlog('hiliting', settings.hilite);
                    page.evaluate(hilite, re, false, hiliteColor, textColor);
                }
            }
            setTimeout(function(){
                vlog('starting render', settings.url);
                page.render( screenshots + tempFile );
                vlog('done render', settings.url);
                page.close();
                if( fs.exists( screenshots + tempFile ) ){
                    vlog('rendered', settings.url, 'to', tempFile);
                    jsonResponse(response, {
                        url: settings.url,
                        fileName: tempFile
                    });
                } else {
                    errorResponse( response, 'could not render url', 500 );
                }
            }, settings.delay * 1000 );
        } else {
            errorResponse( response, 'url failed to load' );
            page.close();
        }
    });
}

var service = server.listen(port, function( request, response ){
    var settings = processArguments(request.url);
    
    if( settings.url ){
        switch( settings.format ){
            case 'PDF':
            case 'PNG':
            case 'JPEG':
                makeScreenshot( settings, response );

                break;
            default:
                errorResponse( response, 'unrecognized format' );
        }
    } else if ( settings.fileName ){
        vlog('screenshot image request', settings.fileName);
        switch( settings.fileName.split('.').pop().toUpperCase() ){
            case 'PDF':
            case 'PNG':
            case 'JPEG':
                fileResponse( response, settings.fileName );
                break;
            default:
                vlog( 'bad file type', settings.fileName);
                errorResponse(response, "file not found", 404);
        }
    } else {
        errorResponse(response);
    }
});

// call makeScreenshot once to give us a chance to bypass BlueHost
makeScreenshot( processArguments('?url=google.com') );