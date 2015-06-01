# Nodejs design patterns

These are my notes from reading the book [Node.js Design Patterns](https://www.packtpub.com/web-development/nodejs-design-patterns)

## Async and sync

### Continuation passing style
Sync using callback:

	function add(a, b, callback){
		callback(a + b);
	}

Async using callback function:

	function add(a, b, callback){
		setTimeout(function(){
			callback(a + b);
		}, 100);
	}

### Direct style (Non-continuation passing style)
Sync usign return

    function add(a, b){
	    return a + b;
    }
    
### Sync or Async?
#### Unpredictable function
	var fs = require('fs');
	var cache = {};
	function inconsistentRead(filename, callback){
		if(cache[filename]){
			// invoked sync
			return callback(null, cache[filename]);
		}
		
		// async function
		fs.readFile(filename, 'utf8', function(err, body){
			if(err){
				return callback(err);
			}
			
			cache[filename] = body;
			callback(null, body);
		});
	}
Problem is with sync call from cache, so callback is invoked before any futher listeners binded on.

#### Fix problem by sync API
	var fs = require('fs');
	var cache = {};
	function consistentReadSync(filename){
		if(cache[filename]){
			return cache[filename];
		}
		
		cache[filename] = fs.readFileSync(filename, 'utf8');
		return cache[filename];
	}
This fix problem by using synch BLOCKING api. Better will be use purely async.

#### Fix problem by scheduling sync callback to be executed "in the future"
	var fs = require('fs');
	var cache = {};
	function consistentRead(filename, callback){
		if(cache[filename]){
			// invoked async
			process.nextTick(function(){
				callback(null, cache[filename]);
			});
		}
		
		// async function
		fs.readFile(filename, 'utf8', function(err, body){
			if(err){
				return callback(err);
			}
			
			cache[filename] = body;
			callback(null, body);
		});
	}

## Async control flow patterns
### Callback hell
	function spider(url, callback){
		var filename = utilities.urlToFilename(url);
		fs.exists(filename, function(exists){
			if(!exists){
				console.log('Downloading ' + url);
				request(url, function(err, response, body){
					if(err){
						callback(err);
					}else{
						mkdirp(path.dirname(filename), function(err){
							if(err){
								callback(err);
							}else{
								fs.writeFile(filename, body, function(err){
									if(err){
										callback(err);
									}else{
										callback(null, filename, true);
									}
								});
							}
						});
					}
				});
			}else{
				callback(null, filename, false);
			}
		});
	}

#### Solution
1. Refactor error checking pattern:
	
		if(err){
			callback(err);
		}else{
			// code to be execute when there are no errors
		}
change to:
	
		if(err){
			return callback(err); // don't forget add "return" to exit execution
		}
		// code to be execute when there are no errors

2. Identify reusable blocks of code and break it to separate functions
 
		function spider(url, callback){
			var filename = utilities.urlToFilename(url);
			
			fs.exists(filename, function(exists){
				if(exists){
					return callback(null, filename, false);
				}
				
				console.log('Downloading ' + url);
				
				download(url, filename, function(err){
					if(err){
						return callback(err);
					}
					
					callback(null, filename, true);
				});
			});
		}
	
		function download(url, filename, callback){
			request(url, function(err, response, body){
				if(err){
					return callback(err);
				}
					
				console.log('Downloaded');
				
				saveFile(filename, body, function(err){
					if(err){
						return callback(err);
					}
					
					console.log('Saved');
					
					callback(null);
				});
			});
		}
		
		function saveFile(filename, contents, callback){
			mkdirp(path.dirname(filename), function(err){
				if(err){
					return callback(err);
				}
		
				fs.writeFile(filename, contents, callback);
			});
		}

### Sequential execution and iteration
	function iterate(index){
		if(tasks.length === index){
			return final();
		}
		var task = tasks[index];
		task(function(){
			iterate(index + 1);
		});
	}
	
	function final(){
		// iteration completed
	};
	
	iterate(0);

Example spider:

	function spider(url, nesting, callback){
		var filename = utilities.urlToFilename(url);
		
		fs.readFile(filename, 'utf8', function(err, body){
			if(err){
				if(err.code !== 'ENOENT'){
					return callback(err);
				}
				
				console.log('Downloading ' + url);
				
				return download(url, filename, function(err, body){
					if(err){
						return callback(err);
					}
					
					spiderLinks(url, body, nesting, callback);
				});
			}
			
			spiderLinks(url, body, nesting, callback);
		});
	}
	
	function spiderLinks(currentUrl, body, nesting, callback){
		if(nesting === 0){
			return process.nextTick(callback);
		}
		
		var links = utilities.getPageLinks(currentUrl, body);
		
		function iterate(index){
			if(index === links.length){
				return callback();
			}
			
			spider(links[index], nesting - 1, function(err){
				if(err){
					return callback(err);
				}
				
				iterate(index + 1);
			});
		}
		
		iterate(0);
	}

### Parallel execution
	var tasks = [...];
	var completed = 0;
	tasks.forEach(function(task){
		task(function(){
			if(++completed === task.length){
				final();
			}
		});
	});
	
	function final(){
		// all tasks completed
	};

Example spider:
		
	function spider(url, nesting, callback){
		
		var filename = utilities.urlToFilename(url);
		
		fs.readFile(filename, 'utf8', function(err, body){
			if(err){
				if(err.code !== 'ENOENT'){
					return callback(err);
				}
				
				console.log('Downloading ' + url);
				
				return download(url, filename, function(err, body){
					if(err){
						return callback(err);
					}
					
					spiderLinks(url, body, nesting, callback);
				});
			}
			
			spiderLinks(url, body, nesting, callback);
		});
	}
	
	function spiderLinks(currentUrl, body, nesting, callback){
		if(nesting === 0){
			return process.nextTick(callback);
		}
		
		var links = utilities.getPageLinks(currentUrl, body);
		
		if(links.length === 0){
			return process.nextTick(callback);	
		}
		
		var completed = 0, errored = false;
		
		function done(err){
			if(err){
				errored = true;
				return callback(err);
			}
			
			if(++completed === links.length && !errored){
				return callback();
			}
		}
		
		links.forEach(function(link){
			spider(link, nesting - 1, done);
		});
	}

There are one race condition. When spider task is invoked async and concurrently fs.readFile is called more times for one file and so file is downloaded multiple times.

####Solution:
		
	var spidermap = {};
	
	function spider(url, nesting, callback){
		if(spidermap[url]){
			return process.nextTick(callback);
		}
		
		spidermap[url] = true;
		...
	}
This prevents spidering one url multiple times.