

const puppeteer = require('puppeteer-extra');
var fs  = require('fs');
var md5 = require('js-md5');
const { exec } = require('child_process');
var axios = require('axios');
const ftpb = require("basic-ftp")
const { Sequelize, Op } = require('sequelize');


const sequelize = new Sequelize(******************************************************************************************
}); 

sequelize.authenticate().then(() => {
	console.log('Connection has been established successfully.');
	}).catch((error) => {
	console.error('Unable to connect to the database: ', error);
});



var headers = {};
var cookies = [];

const client = new ftpb.Client()
client.ftp.verbose = true

const myDir = "/var/www/www-root/data/www/www.fitlead.ru/image/"; 
const pathSavePhoto = "catalog/tovar";
 
var sanProduct = sequelize.import("./models/san_product.js"); 
var sanCategoryDescription = sequelize.import("./models/san_category_description.js");
var tableBlacklist = sequelize.import("./models/table_blacklist.js");
var sanCategory = sequelize.import("./models/san_category.js");
var sanCategoryToLayout = sequelize.import("./models/san_category_to_layout.js");
var sanCategoryToStore = sequelize.import("./models/san_category_to_store.js");
var sanCategoryPath = sequelize.import("./models/san_category_path.js");
var sanAttributeGroup = sequelize.import("./models/san_attribute_group.js");
var sanAttributeGroupDescription = sequelize.import("./models/san_attribute_group_description.js");
var sanAttributeDescription = sequelize.import("./models/san_attribute_description.js");
var sanAttribute = sequelize.import("./models/san_attribute.js");
var sanManufacturer = sequelize.import("./models/san_manufacturer.js");
var sanManufacturerDescription = sequelize.import("./models/san_manufacturer_description.js");
var sanManufacturerToStore = sequelize.import("./models/san_manufacturer_to_store.js");
var sanProductToStore = sequelize.import("./models/san_product_to_store.js");
var sanProductToLayout = sequelize.import("./models/san_product_to_layout.js");
var sanProductToCategory = sequelize.import("./models/san_product_to_category.js");
var sanProductTab = sequelize.import("./models/san_product_tab.js");
var sanProductTabDesc = sequelize.import("./models/san_product_tab_desc.js");
var sanProductImage = sequelize.import("./models/san_product_image.js");
var sanProductDescription = sequelize.import("./models/san_product_description.js");
var sanSeoUrl = sequelize.import("./models/san_seo_url.js");
var sanProductAttribute = sequelize.import("./models/san_product_attribute.js");

const StealthPlugin = require('puppeteer-extra-plugin-stealth')
//const stealth = StealthPlugin()
//stealth.enabledEvasions.delete('user-agent-override')
//puppeteer.use(stealth)
puppeteer.use(StealthPlugin());

const blackList = [
	{
		category: 'Душевые кабины',
		brands: ['Am.Pm','AM.PM']
	},
	{
		category: 'Душевые кабины, ограждения', 
		brands: ['Am.Pm','AM.PM']
	} 
]

const translit = str => {
	//console.log(str);
    if(str != null){
        str = str.toLowerCase().replace(/[^а-яё0-9a-z ]/g, '').replace(/ +/g, ' ').trim();
        let letters = {
			'а': 'a', 'б': 'b', 'в': 'v', 'г': 'g', 'д': 'd', 
			'е': 'e', 'ё': 'e', 'ж': 'j', 'з': 'z', 'и': 'i', 
			'к': 'k', 'л': 'l', 'м': 'm', 'н': 'n', 'о': 'o', 
			'п': 'p', 'р': 'r', 'с': 's', 'т': 't', 'у': 'u', 
			'ф': 'f', 'х': 'h', 'ц': 'c', 'ч': 'ch', 'ш': 'sh', 
			'щ': 'shch', 'ы': 'y', 'э': 'e', 'ю': 'u', 'я': 'ya',
			' ': '-',
		}, newStr = [];
		str = str.replace(/[ъь]+/g, '').replace(/й/g, 'i')
		for ( var i = 0; i < str.length; ++i ) {
            if(letters[ str[i] ] != undefined){
                newStr.push(letters[ str[i] ]);
				} else {
                newStr.push(str[i]);
			}
            
		} 
        return newStr.join('');
	}
}

////***переворачиваем код***////

const reverseStr = str => {
    
    if(str) return str.split("").reverse().join("");
    else return "";
    
    
}

////**********end***********////

////---проверяем есть ли категория в черном списке---////

const checkBlackList = async name => {
    
    //console.log('Проверяем черный список '+name);
    
    const result = await tableBlacklist.findOne({where: {name: name}});
    
    if(result) return true;
    
    return false;
}

////-----------------------end-----------------------////

///---получаем id category если она есть---////

const getCategoryId = async name => {
    
    const result = await sanCategoryDescription.findOne({attributes: ['category_id',],where: {name: name}, raw: true});
    
    if(result) return result.category_id;
    
    return false;
}

////-----------------------end-----------------------////

////***кладем товар в базу - тут несколько функций***////

////---работа с категориями ---////

const insertCategory = async categories => {
    
    var parent_id = 0;
    var path = [];
    console.log('categoriescategoriescategoriescategoriescategories   ',categories);
    for(name of categories){ 
		
        if(name === 'Каталог') continue;
        
        // проверяем категорию в черном списке
        var ch = await checkBlackList(name);
        
        if(ch){
            console.log('Категория '+name+' в черном списке');
            return false;
			} else {
            // работаем если категорию можно парсить
            
            //console.log('Категории '+name+' можно парсить');
            
            var category_id = await getCategoryId(name);
            
            //console.log('category_id',category_id);
			
            if(!category_id){
                
                console.log('Категории '+name+' у нас нет. Вставляем');
                
                const res = await sanCategory.create({ 
                    parentId: parent_id, 
                    top: 0,
                    column: 1,
                    status: 1,
                    dateAdded: sequelize.literal('CURRENT_TIMESTAMP'),
                    dateModified: sequelize.literal('CURRENT_TIMESTAMP'),
                    parscontent: 1                                 
				});
                
                //console.log('res',res.categoryId);
                category_id = res.categoryId;
                
                await sanCategoryDescription.create({ 
                    categoryId: category_id, 
                    languageId: 1,
                    name: name,
                    description: '&lt;p&gt;&lt;br&gt;&lt;/p&gt;',
                    metaTitle: name,
                    metaH1: name,
                    metaDescription: '',
                    metaKeyword: '',
                    parscontent: 1,
                    oldTitle: name,
				});
				var	nameTranslit = translit(name);
				await sanSeoUrl.findOrCreate({where:{
					query: 'category_id='+category_id,
				},
				defaults: {  
					languageId: 1,
					keyword: nameTranslit,
					query: 'category_id='+category_id
					
				}});  
                //console.log('sanCategoryToLayout');
                
                await sanCategoryToLayout.create({ 
                    categoryId: category_id,
                    storeId: 0, 
                    layoutId: 0, 
                    parscontent: 1
				});
                
                //console.log('sanCategoryToStore');
                
                await sanCategoryToStore.create({ 
                    categoryId: category_id,
                    storeId: 0, 
                    parscontent: 1
				});
                
			} else console.log('Категория '+name+' у нас есть. Идем дальше');
            
            parent_id = category_id;
            path.push(category_id);
            
			}
			
			//console.log('name',name);
			
		}
		
		index = 0;
		for(item of path) {
			
			console.log('sanCategoryPath1');
			
			await sanCategoryPath.findOrCreate({
				where:{
					categoryId: parent_id,
					pathId: item,
				},
				defaults: { 
					categoryId: parent_id,
					pathId: item, 
					level: index, 
					parscontent: 1
				}}); 
				
				console.log('sanCategoryPath2');
				await sanCategoryPath.findOrCreate({
					where:{
						categoryId: item,
						pathId: item,
					},
					defaults: { 
						categoryId: item,
						pathId: item, 
						level: index,  
						parscontent: 1
					}});
					
					let n = index;
					while(true){
						if(n == 0)
						{break;}
						
						n--;
						
						console.log('sanCategoryPath3');
						
						await sanCategoryPath.findOrCreate({
							where:{
								categoryId: item,
								pathId: path[n], 
							},
							defaults: { 
								categoryId: item,
								pathId: path[n], 
								level: n, 
								parscontent: 1
							}});
							
					}
					index++;
		}; 
		
		console.log('INSERT CATEGORY FINISH');
		
		
		return true;
		
	}
	
	//тест
	//insertCategory(['Plitka','Plitka na pol','Plitka na pol white and strong'])
	
	////-------end-----------------////
	
	////---работа с атрибутами  ---////
	
	const findAttribute = async name => {
		
		const result = await sanAttributeDescription.findOne({attributes: ['attribute_id',],where: {name: name}, raw: true});
		
		return result;
		
		
	}
	
	const insertAttribute = async attributes => {
		
		let maxId = await sanAttributeGroup.max('sort_order');
		//console.log('maxId 9999', maxId);
		
		//console.log('attributesattributesattributes 55555', attributes);
		
		//console.log('attributes.group attributes.group 44444', attributes.group); 
		//console.log('attributesgroup 33333333', attributes['group']); 
		
		//  for(attribute of attributes['props']){  цикл нужен, если у нас несколько групп атрибутов
		
		const result = await sanAttributeGroupDescription.findOne({attributes: ['attribute_group_id',],where: {name: attributes['group']}, raw: true});
		let attributeGroupId = false;
		//console.log('result 3377777777777777777', result); 
		if(!result){
			
			maxId++;
			
			const res = await sanAttributeGroup.create({ 
				sortOrder: maxId,
				parscontent: 1, 
			});
			
			attributeGroupId = res.attributeGroupId;
			
			await sanAttributeGroupDescription.create({ 
				attributeGroupId: attributeGroupId,
				languageId: 1, 
				name: attributes['group'],
				parscontent: 1,
				oldTitle: attributes['group'],
			});  
			console.log('вставили группу атрибутов',attributes['group']);
			
			} else {
			attributeGroupId = result.attribute_group_id;
			console.log('уже есть такая группа атрибутов',attributes['group']);
		} 
		
		//	console.log('attributes.props 1111111', attributes.props); 
		for(prop of attributes.props){
			
			//	console.log('prop.prop 9999999',prop.prop);
			var propReadtyprop = prop.prop[0];
			const result = await findAttribute(propReadtyprop);
			//	console.log('propReadtyprop 4000000000',propReadtyprop);
			if(!result){
				
				const res = await sanAttribute.create({ 
					attributeGroupId: attributeGroupId,
					sortOrder: 10,
					parscontent: 1,
				});
				
				let attributeId = res.attributeId;
				
				await sanAttributeDescription.create({ 
					attributeId: attributeId,
					languageId: 1, 
					name: propReadtyprop,
					parscontent: 1,
					oldTitle: propReadtyprop,
				}); 
				//	console.log('вставили атрибут',propReadtyprop);
				
			} 
			
		}
		
		// } конец цикла
		
	}
	
	//insertAttribute([{group: 'plitka', props: [{ prop: 'tipp', value: 'электрический' }]}]);
	
	////-------end-----------------////
	
	////---работа с производителями ---////
	
	const insertBrand = async (brand) => {
		
		const result = await sanManufacturer.findOne({attributes: ['manufacturer_id',],where: {oldTitle: brand}, raw: true});
		let manufacturerId = false;
		if(!result){
			
			console.log('новый бренд', brand);
			
			const res = await sanManufacturer.create({ 
				name: brand,
				sortOrder: 10,
				parscontent: 1, 
				oldTitle: brand,
			});
			
			manufacturerId = res.manufacturerId;
			
			await sanManufacturerDescription.create({
				manufacturerId: manufacturerId,
				name: brand,
				languageId: 1, 
				description: '&lt;p&gt;&lt;br&gt;&lt;/p&gt;',
				metaTitle: brand,
				metaH1: brand,
				metaDescription: '',
				metaKeyword: '',
				parscontent: 1,
			});
			
			await sanManufacturerToStore.create({
				manufacturerId: manufacturerId,
				storeId: 0,
				parscontent: 1,
			});
			
			} else {
			
			//console.log('бренд уже есть ', brand);
			manufacturerId = result.manufacturer_id;
		}
		
		return manufacturerId;
		
		
	}
	//insertBrand('plitka');
	
	////-------end---------------------////
	
	////---работа с товаром ---////
	
	const insertProduct = async (data) => {
		
		console.log('Вставляем новый товар');
		
		//console.log('data.optiondata.optiondata.optiondata.optiondata.option', data.option);
		try{
			
			if(data.nameProduct.includes('Уценка') || data.nameProduct.includes('уценка')) return false;
			
			if(!insertCategory(data.categories)) return false;
			if(data.option){
				await insertAttribute(data.option);
			};
			
			//	let manufacturerId = await insertBrand(data.brand);
			
			const res = await sanProduct.findOrCreate({where:{
				sku: data.sku,
			},
			defaults: { 
				donorUrl: data.donorUrl,
				model: data.model,
				sku: data.sku,
				upc: '',
				ean: '',
				jan: '',
				isbn: '', 
				mpn: '',
				location: '',
				quantity: 10000,
				stockStatusId: 7, 
				image: '',
				//manufacturerId: manufacturerId,
				manufacturerId: '',
				price: data.price,
				taxClassId: 0, 
				dateAvailable: sequelize.literal('CURRENT_TIMESTAMP'),
				status: 1,
				dateAdded: sequelize.literal('CURRENT_TIMESTAMP'),
				dateModified: sequelize.literal('CURRENT_TIMESTAMP'), 
				actual: 1,
				parscontent: 1,
				sortOrder: 1000,
				updateDt: 0
			}});
			
			let productId = res[0].productId;
			console.log(productId, '  ++++++++++++++ productId  ++++++++++++++++++++++++');
			
			
			await sanSeoUrl.findOrCreate({where:{
				query: 'product_id='+productId,
			},
			defaults: {  
				languageId: 1,
				keyword: data.nameProductTranslit,
				query: 'product_id='+productId
				
			}});
			
			await sanProductDescription.findOrCreate({where:{
				productId: productId,
			},
			defaults: { 
				productId: productId,
				languageId: 1,
				name: data.nameProduct,
				description: data.description,
				shortDescription: '',
				tag: '',
				metaTitle: '', 
				metaH1: '',
				metaDescription: '',
				metaKeyword: '',
				parscontent: 1, 
				oldTitle: data.nameProduct,
			}});
			if(data.option){
				for(prop of data.option.props){
					//	console.log('++++++++++++ prop 502 строка +++++++++++++++ ', prop);
					
					//	console.log('++++++++++++ prop.prop 7777777 строка +++++++++++++++ ', prop.prop);
					var propReadtyprop = prop.prop[0];
					var valueReadtyvalue = prop.value[0];
					const result = await findAttribute(propReadtyprop);
					
					await sanProductAttribute.findOrCreate({
						where:{
							productId: productId,
							attributeId: result.attribute_id, 
						},
						defaults: { 
							productId: productId,
							attributeId: result.attribute_id, 
							languageId: 1, 
							text: valueReadtyvalue,
							parscontent: 1, 
						}});
						
				}
				
			}
			
			let c = 0;
			for(pic of data.pic){
				
				c++;
				
				if(c == 1){
					
					let product = await sanProduct.findOne({where:{productId: productId}});
					
					product.update({
						image: pic
					})
					
					}else{
					
					await sanProductImage.findOrCreate({
						where:{
							image: pic,
						},
						defaults: { 
							productId: productId,
							image: pic, 
							sortOrder: c,
							video: '',
							parscontent: 1, 
						}});
						
				}
				
			}
			
			categoryId = await getCategoryId(data.categories[data.categories.length - 1]);
			
			await sanProductToCategory.findOrCreate({
				where:{
					productId: productId,
					categoryId: categoryId,
				},
				defaults: { 
					productId: productId,
					categoryId: categoryId,
					mainCategory: 1,
					parscontent: 1, 
				}});
				
				await sanProductToLayout.findOrCreate({
					where:{
						productId: productId,
					},
					defaults: { 
						productId: productId,
						storeId: 0,
						layoutId: 0,
						parscontent: 1, 
					}});
					
					await sanProductToStore.findOrCreate({
						where:{
							productId: productId,
						},
						defaults: { 
							productId: productId,
							storeId: 0,
							parscontent: 1, 
						}});
						
						console.log('Внесен товар ',productId, data.code);
						
						} catch (e) {
						console.log('ERROR',e)
		}
		
	}
	
	////-------end-------------////
	
	///******************end*****************************////
	
	////***открываем браузер и берем инфу***////
	
	const proxy = "((((((((((((((";
	
	const getPage = async (url, mode, pathSaveImg = null) => {
		
		var response = '';
		
		console.log('url 630 cтрока ',url);
		
		
		try{
			const browser = await puppeteer.launch({ headless: true, args: ['--no-sandbox', '--enable-blink-features=HTMLImports', '--proxy-server=http://'+proxy] });
			//const browser = await puppeteer.launch({ headless: true, args: ['--no-sandbox'] });
			
			const page = await browser.newPage();
			
			const clientP = await page.target().createCDPSession();
			
			await clientP.send('Network.enable', {
				maxResourceBufferSize: 1024 * 1204 * 100,
				maxTotalBufferSize: 1024 * 1204 * 200,
			});
			/*await page.authenticate({ 
				username: 'mnujaz' , 
				password:'FCaOLp1gTe' 
			});*/
			//await page.setUserAgent('Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3987.0 Safari/537.36');
			/*await page._client.send('Network.enable', {
				maxResourceBufferSize: 1024 * 1204 * 100,
				maxTotalBufferSize: 1024 * 1204 * 200,
			})*/
			/*await page.goto('https://bot.sannysoft.com')
				
				await page.setViewport({ width: 1024, height: 768 });
				//await page.goto(url);
				await page.waitFor(5000)
				await page.screenshot({ path: 'testresult.png', fullPage: true });
				await browser.close();
				console.log(`All done, check the screenshot. ✨`);
			return false;*/
			
			
			/*await page.setRequestInterception(true);
				page.on('request', request => {
				if (request.resourceType() === 'image' || request.resourceType() === 'stylesheet')
				request.abort();
				else
				request.continue();
			});*/
			// await page.setViewport({ width: 1024, height: 768 });
			
			//while(true){
			
			
			var viewSource = await page.goto(url, {
				waitUntil: 'networkidle2',
				// Remove the timeout
				timeout: 0
			}) 
			
			/*(mode === 'getData'){
				
				cookies = await page.cookies()
				headers = viewSource.headers();
				console.log('headers',headers);
			} */
			
			//await page.waitFor(3000);
			await page.screenshot({ path: 'screenshot.png', fullPage: true });
			//const html = await page.content();
			
			
			if(mode === 'firstPage'){ // самая первая страница
				
				
				console.log('firstPage', new Date());
				
				var listUrls = [];
				
				const html = await page.content();
				
				var linksCount = (await page.$$('.image_wrapper_block > a')).length +1;
				console.log('Считаем количество ссылок на товары на странице категорий плюс один  -  mode === firstPage ', linksCount); 
				
				if(!linksCount || linksCount.length < 1){
					await browser.close();
					console.log('на странице нет ссылок, возможно поменялись теги на доноре на странице категорий');
					return false;
				}
				// Перебор и запись всех статей на выбранной странице
				for (i = 1; i < linksCount; i++) { 
					var querySelectBlock = '#right_block_ajax > div.inner_wrapper > div.ajax_load.cur.block > div.js_wrapper_items > div > div.catalog_block.items.row.margin0.has-bottom-nav.js_append.ajax_load.block.flexbox > div:nth-child('+i+')';
					
					//relativeLinksInBlock + a получаем ссылку
					var relativeLinksInBlock = querySelectBlock + ' a';
					var relativeLinks = await page.$eval(relativeLinksInBlock, e => e.getAttribute('href')); 
					
					// Превращение ссылок в абсолютные               
					var relativeLinks = relativeLinks.replace('/', 'https://zakaz-sport.ru/');
					
					console.log('абсолютные ссылки ', relativeLinks);
					
					//iDview получаем Айдишку
					
					var iDview = await page.$eval(querySelectBlock, e => e.getAttribute('data-id')); 
					console.log('Айдишники ', iDview);
					
					//загоняем ссылки в массив listUrls если нет такой sku
					const product = await sanProduct.findOne({where: {sku: iDview}}); 
					
					if(!product){
						listUrls.push(relativeLinks);
						console.log('Внесена ссылка ',relativeLinks);
					}
					else {
						// то есть если товар есть, обновляем цену со страницы Категорий
						var priceBlock = querySelectBlock + ' .price';
						var price = await page.$eval(priceBlock, e => e.getAttribute('data-value')); 
						
						
						if(price){
							
							// let price = it.priceOld ? it.priceOld : it.price;
							// console.log('price ', price);
							console.log('Обновляем цену товар '+relativeLinks+' цена '+price);
							
							const result3 = await product.update({
								donorUrl:  relativeLinks,
								price: price,
								actual: 1,
								status: 1,
								dateModified: sequelize.literal('CURRENT_TIMESTAMP'),
							});
							} else {
							console.log('Для товара нет цены '+relativeLinks);
							const result3 = await product.update({
								donorUrl:  relativeLinks,
								actual: 0,
								status: 0,
								dateModified: sequelize.literal('CURRENT_TIMESTAMP'),
							});  
						}
					}
					
				};  
				let count = 0;
				
				var pagination = await page.$eval('.element-count', e => e.innerHTML).catch(error => error.toString() !== 'Error2');
				if(pagination && pagination !== true){
					count = Math.ceil(pagination.replace(/\D/g,'') / 20);
				}
				
				console.log('count',count);
				
				let pageTemp = url+'?PAGEN_1=';
				
				//console.log('pageTemp',pageTemp);
				console.log('listUrls',listUrls);
				
				response = {listUrls: listUrls, count: count, pageTemp: pageTemp};
				
				
				} else if(mode == 'nextPage'){ // каждая следующая страница
				
				var listUrls = []; // переменная которая содержит ссылки на новые товары
				
				const html = await page.content();
				
				var linksCount = (await page.$$('.image_wrapper_block > a')).length +1;
				console.log('Считаем количество ссылок на товары на странице категорий плюс один  -  mode === nextPage', linksCount); 
				
				if(!linksCount || linksCount.length < 1){
					await browser.close();
					console.log('на странице нет ссылок, возможно поменялись теги на доноре на странице категорий  -  mode === nextPage');
					return false;
				}
				// Перебор и запись всех статей на выбранной странице
				for (i = 1; i < linksCount; i++) { 
					var querySelectBlock = '#right_block_ajax > div.inner_wrapper > div.ajax_load.cur.block > div.js_wrapper_items > div > div.catalog_block.items.row.margin0.has-bottom-nav.js_append.ajax_load.block.flexbox > div:nth-child('+i+')';
					
					//relativeLinksInBlock + a получаем ссылку
					var relativeLinksInBlock = querySelectBlock + ' a';
					var relativeLinks = await page.$eval(relativeLinksInBlock, e => e.getAttribute('href')); 
					
					// Превращение ссылок в абсолютные               
					var relativeLinks = relativeLinks.replace('/', 'https://zakaz-sport.ru/');
					
					console.log('абсолютные ссылки ', relativeLinks);
					
					
					//iDview получаем Айдишку
					
					var iDview = await page.$eval(querySelectBlock, e => e.getAttribute('data-id')); 
					console.log('Айдишники ', iDview);
					
					//загоняем ссылки в массив listUrls если нет такой sku
					const product = await sanProduct.findOne({where: {sku: iDview}}); 
					
					if(!product){
						listUrls.push(relativeLinks);
						console.log('Внесена ссылка ',relativeLinks);
						}else {
						// то есть если товар есть, обновляем цену со страницы Категорий
						var priceBlock = querySelectBlock + ' .price';
						var price = await page.$eval(priceBlock, e => e.getAttribute('data-value')); 
						
						
						if(price){
							
							// let price = it.priceOld ? it.priceOld : it.price;
							// console.log('price ', price);
							console.log('Обновляем цену товар '+relativeLinks+' цена '+price);
							
							const result3 = await product.update({
								donorUrl:  relativeLinks,
								price: price,
								actual: 1,
								status: 1,
								dateModified: sequelize.literal('CURRENT_TIMESTAMP'),
							});
							} else {
							console.log('Для товара нет цены '+relativeLinks);
							const result3 = await product.update({
								donorUrl:  relativeLinks,
								actual: 0,
								status: 0,
								dateModified: sequelize.literal('CURRENT_TIMESTAMP'),
							});  
						}
					}
					
				};  
				response = {listUrls: listUrls}
				
				} else if(mode == 'getData'){ //получаем данные со страницы и заносим в базу
				
				console.log('получаем данные со страницы 781 стр  -  mode === getData ',url);
				
				const html = await page.content();
				
				var donorUrl = url;
				
				var linksCount = (await page.$$('.image_wrapper_block > a')).length +1;
				console.log('Считаем количество ссылок на товары на странице категорий плюс один -  mode === getData', linksCount); 
				
				if(!linksCount || linksCount.length < 1){
					await browser.close();
					console.log('на странице нет ссылок, возможно это не страница категорий mode === getData');
					return false;
				}
				// Перебор и запись всех статей на выбранной странице
				for (i = 1; i < linksCount; i++) { 
					var querySelectBlock = '#right_block_ajax > div.inner_wrapper > div.ajax_load.cur.block > div.js_wrapper_items > div > div.catalog_block.items.row.margin0.has-bottom-nav.js_append.ajax_load.block.flexbox > div:nth-child('+i+')';
					
					//relativeLinksInBlock + a получаем ссылку
					var relativeLinksInBlock = querySelectBlock + ' a';
					var relativeLinks = await page.$eval(relativeLinksInBlock, e => e.getAttribute('href')); 
					
					// Превращение ссылок в абсолютные               
					var relativeLinks = relativeLinks.replace('/', 'https://zakaz-sport.ru/');
					
					console.log('абсолютные ссылки ', relativeLinks);
					
					//загоняем ссылки в массив listUrls
					listUrls.push(relativeLinks);
					
				};  
				try{
					var nameProduct =  await page.$eval('h1', e => e.innerHTML); 
					//	console.log('nameProduct ',nameProduct); 
					} catch (err){
					var nameProduct = ''; 
				}
				
				var nameProductTranslit = translit(nameProduct);
				try{
					var categories =  await page.$eval('#bx_breadcrumb_2', e => e.innerText);// console.log('categories ',categories); 
					} catch (err){
					var categories = '';
				}
				try{
					var model = await page.$eval('.article__value', e => e.innerHTML); 
					} catch (err){
					var model = ''; 
				}
				try{
					var price = await page.$eval('.price_matrix_wrapper .price', e => e.getAttribute('data-value')); 
					} catch (err){
					var price = '';
				}
				try{
					var sku = await page.$eval('.in-cart', e => e.getAttribute('data-item')); 
					} catch (err){
					var sku = '';
				}
				//var price = await price_matrix_wrapper.evaluate(el => el.getAttribute('data-value'));
				
				//	var oldPrice = ob.CardPrice.data.oldPrice; 
				try{
					var description = await page.$eval('.detail-text-wrap', e => e.innerHTML);
					} catch (err){
					var description = '';
				}
				var code ='';
				var brand ='';
				
				// берем аттрибуты 
				
				const propsTr = await page.$$('#props tr.js-prop-replace');
				console.log(propsTr.length, 'propsTr 8888888888');
				if(propsTr.length > 0){
					var props = [];
					var group = 'Характеристики';  
					for (let i = 0; i < propsTr.length; i++) {
						var propsName = await propsTr[i].$eval('.char_name span', e => e.innerText);  
						var propsNameReady = propsName.toString().replace(/\t/g, '').split('\r\n');
						var propsValue = await propsTr[i].$eval('.char_value span', e => e.innerText); 
						var propsValueReady = propsValue.toString().replace(/\t/g, '').split('\r\n');
						
						props.push({prop:propsNameReady, value:propsValueReady});	 
					}	
					var option = {group: group, props: props};
					}else{
					var option = '';
				}
				
				console.log('option ',option);
				
				
				//console.log(option, '========= option33333333333333333 ===============');
				////----Забираем фотографии----////sole.log(props); */
				
				var picsrc = [];
				var pic = [];
				const elements = await page.$$('.product-detail-gallery__item img');
				for (let i = 0; i < elements.length; i++) {
					var srcDataBig = await elements[i].evaluate(el => el.getAttribute('src'));
					// Превращение ссылок на картинки в абсолютные               
					srcDataBig = srcDataBig.replace('/', 'https://zakaz-sport.ru/');
					picsrc.push(srcDataBig);
				}
				//	console.log(picsrc);
				
				
				
				
				
				let catalog = translit(categories);
				console.log('catalog  888888 ',catalog);
				console.log('Забираем фотографии');
				
				let ensureDir2 = await client.ensureDir( "fitlead.ru/public_html/image/catalog/tovar/"+catalog+"/"+reverseStr(sku));
				console.log('ensureDir2  500000 ',ensureDir2);
				await client.cd("/");
				
				const writeFilePic = (response, pathSave) => {
					return new Promise((resolve, reject) => {
						
						try{
							const pf = response.data.pipe(fs.createWriteStream(pathSave))
							pf.on('finish', async () => {
								var stats = fs.statSync(pathSave);
								var fileSizeInBytes = stats.size;
								
								resolve(fileSizeInBytes);
							})
							} catch (err){
							reject(err)
						}
						
					})
				}  
				
				const optimPhoto = (pathSave) => {
					return new Promise((resolve, reject) => {
						exec(`python3 /root/parsSant2/parser.santekhnika-darom.ru/pars2/puppeter/opt.py "${pathSave}"`, (err, stdout, stderr) => {
							if (err) {
								//some err occurred
								reject(err)
								} else {
								// the *entire* stdout and stderr (buffered)
								//console.log(`stdout: ${stdout}`);
								//console.log(`stderr: ${stderr}`);
								resolve(true);
								console.log('Оптимизация завершена - заливаем на фтп')
							}
						});
					})
				}
				console.log('picsrc 333333333 ', picsrc);
				 
				if(picsrc && picsrc.length > 0){
					
					for (let i = 0; i < picsrc.length; i++) {
						
						console.log('picsrc 4444656456 ' , picsrc[i]);
						let name = picsrc[i].split("/")[picsrc[i].split("/").length - 1];
						console.log('name  111111 ' , name);
						let pathSave = "/root/parsSant2/parser.santekhnika-darom.ru/pars2/puppeter/bufferPhotoGym/"+catalog+"/"+reverseStr(sku)+"/"+name
						
						if (!fs.existsSync("/root/parsSant2/parser.santekhnika-darom.ru/pars2/puppeter/bufferPhotoGym/"+catalog)) fs.mkdirSync("/root/parsSant2/parser.santekhnika-darom.ru/pars2/puppeter/bufferPhotoGym/"+catalog, 0777);
						if (!fs.existsSync("/root/parsSant2/parser.santekhnika-darom.ru/pars2/puppeter/bufferPhotoGym/"+catalog+"/"+reverseStr(sku))) fs.mkdirSync("/root/parsSant2/parser.santekhnika-darom.ru/pars2/puppeter/bufferPhotoGym/"+catalog+"/"+reverseStr(sku), 0777);
						
						try {
							
							const response = await axios({
								method: 'GET',
								url: picsrc[i],
								responseType: 'stream',
							})
							
							var fileSizeInBytes = await writeFilePic(response,pathSave);
							console.log('Файл загрузился в папку 4444455555:)',pathSave, fileSizeInBytes);
							
							if(fileSizeInBytes > 100000){
								console.log('Файл большой - нужно оптимизировать')
								await optimPhoto(pathSave);
								await client.uploadFrom(pathSave, "fitlead.ru/public_html/image/catalog/tovar/"+catalog+"/"+reverseStr(sku)+"/"+name); 
								
								
								
								} else {
								console.log('Файлу оптимизация не нужна')
								await client.uploadFrom(pathSave, "fitlead.ru/public_html/image/catalog/tovar/"+catalog+"/"+reverseStr(sku)+"/"+name);
							}
							
							
							
							pic.push(pathSavePhoto+"/"+catalog+"/"+reverseStr(sku)+"/"+name);
							
							} catch (error) {
							
							console.log('!Не получается забрать файл ',picsrc[i], error);
							
						}
						
					}
				}
				
				// Конец забора картинок
				var infoCompany = {
					catalog: catalog,
					donorUrl: donorUrl,
					//categories: categories,
					//	categories: [ 'Каталог', 'Мебель для ванной комнаты', 'Зеркала' ],
					categories: [ categories ],
					nameProduct: nameProduct,
					nameProductTranslit: nameProductTranslit,
					description: description, 
					code: '',
					sku: sku,
					brand: '',
					model: model, 
					price: price, 
					option: option,
					pic: pic 
					
				}
				
				console.log('infoCompany',infoCompany); 
				
				insertProduct(infoCompany); 
				response = '';
				
				} else if(mode === 'getImage'){ //получаем данные со страницы и заносим в базу
				
				
				
				let cookiesStr = "";
				
				cookies.forEach(it => {
					cookiesStr += it.name+"="+it.value+"; "
				})
				
				console.log('cookiesStr',cookiesStr)
				
				cookiesStr = "_lcid=1283; _laddr=%D0%9A%D0%B0%D0%B7%D0%B0%D0%BD%D1%8C; _llocality=%D0%9A%D0%B0%D0%B7%D0%B0%D0%BD%D1%8C; _lcname=%D0%9A%D0%B0%D0%B7%D0%B0%D0%BD%D1%8C; _lregname=%D0%A0%D0%B5%D1%81%D0%BF%D1%83%D0%B1%D0%BB%D0%B8%D0%BA%D0%B0%20%D0%A2%D0%B0%D1%82%D0%B0%D1%80%D1%81%D1%82%D0%B0%D0%BD; PHPSESSID=1a123772f2d18c9667c5c61224b07910; exp_id=ab_0059; exp_var=2; _gid=GA1.2.413065532.1659873659; notFirstVisit=1; BITRIX_SM2_SALE_UID=2111883923; _ym_uid=1659873661776883080; _ym_d=1659873661; cted=modId%3Dpptoqeul%3Bclient_id%3D1823335227.1659873659%3Bya_client_id%3D1659873661776883080; tmr_lvid=c83e0c3462dd31587d065a6ab4c9d87d; tmr_lvidTS=1659873661410; _userGUID=0:l6j9x649:qhRseWQEILMDus_JpoBLaF2BVbeFQZD0; FPID=FPID2.2.GCkom2w7v6t0GeNDwvzuazRFDYFaetefY%2Brc5n220fM%3D.1659873659; _ym_isad=1; uxs_uid=9b89d030-1648-11ed-8a3b-31afd9269bb4; popmechanic_sbjs_migrations=popmechanic_1418474375998%3D1%7C%7C%7C1471519752600%3D1%7C%7C%7C1471519752605%3D1; FPLC=qlz0OB99MZ2MFTvrMdi1r7N42r%2BvI7%2BXVxpzSA6svkNwD8wEC7bnAG2zOMGNvD1WgS%2BeC7YBjAwgdITyAY8%2Fko%2BavYZ7Rcg8B%2B3zJFxMVFJLlKHgFnPSo83UQwuZ6Q%3D%3D; a_user_interesting=%5B4596005%5D; cf_chl_2=5176190e9ab950f; cf_chl_prog=x15; cf_clearance=68V485O_I7Ylj7Na2PYbRvaq1GLtNDylzqKI0gW4gC4-1659904338-0-150; lastView=%5B4596005%2C5535548%5D; __cf_bm=cwpyTYDRWg1pvBLKReFb2Cfo1h.xcZ4gHT_9b0QA2kU-1659905438-0-ATcEeeqsoEmVZbeehrLDRAhJjghrmqVwanUYjueXydC6nEBFbD0hLvZtwK6trC1B6YQD+3ePlAqW8uvZRtwY8ufVe55b/VePSrx/EAJZRYAu13MSZeogbRFCtABFrnvjYoQPVKI0vAEk79I5S3WeUVlkmgwTaUJlLNI8ZBPFoyLh; _ga=GA1.1.1823335227.1659873659; _ga_VCTF7KZ7Y5=GS1.1.1659905441.2.1.1659905441.0; _ga_46LB69QVTN=GS1.1.1659905441.2.1.1659905441.60; tmr_detect=1%7C1659905442451; DIGI_CARTID=33257849846; dSesn=24dcaa21-edb0-42f4-5533-f8c64983e1e3; _dvs=0:l6jsuca2:sF8FpyhDvpT8iuNxIT~Ww0PwTJeLlpCB; _ym_visorc=b; mindboxDeviceUUID=4fe3fb3b-0237-4c39-bf98-fa3b27b572ea; directCrm-session=%7B%22deviceGuid%22%3A%224fe3fb3b-0237-4c39-bf98-fa3b27b572ea%22%7D; tmr_reqNum=27"
				
				
				headers["set-cookie"] = cookiesStr;
				
				console.log('Пробуем загрузить фото',url)
				
				let resp = await axios.get(url, {responseType: 'blob',headers: headers})
				
				console.log('response',resp.data.length)
				
				fs.writeFile(pathSaveImg, resp.data, (err) => {
					if (err) console.log('ОШИБКА СОХРАНЕНИЯ',err)
					console.log('The file has been saved!');
				});
				
				
				//console.log("The file was saved! "+url);
				
				
				}else if(mode === 'updatePrice'){
				console.log('получаем данные со страницы ',url);
				
				const html = await page.content();
				
				////*** забираем json *** ////
				
				var myRe = /var __SD__ = ({.*?}}});/gi;
				var object = myRe.exec(html);
				
				if(!object || object.length < 1){
					await browser.close();
					console.log('Не смог взять товар со старницы регуляркой - пропускаем 861 строка');
					return false;
				}
				
				let ob = JSON.parse(object[1]);
				
				//console.log('Обьект ',ob);
				
				if(!ob || !ob.CardMedia || !ob.CardMedia.data){
					
					console.log('не смог найти данных в json');
					await browser.close();
					response = false;
					} else {
					
					let price = ob.CardPrice.data.price;
					let oldPrice = ob.CardPrice.data.oldPrice;
					let status = ob.CardPrice.data.status;
					
					const product = await sanProduct.findOne({where: {donorUrl: url}});
					
					if(price && status !== "unavailable"){
						
						console.log('price',price);
						
						if(oldPrice) price = oldPrice;
						
						const result3 = await product.update({
							price: price,
							actual: 1,
							status: 1,
							dateModified: sequelize.literal('CURRENT_TIMESTAMP'),
						});
						} else {
						console.log('Товар не актуален '+url);
						const result3 = await product.update({
							actual: 0,
							status: 0,
							dateModified: sequelize.literal('CURRENT_TIMESTAMP'),
						});  
					}
					
				}
				} else {
				response = false;
			}
			
			//console.log(html);
			await browser.close();
			
			} catch (e) {
			
			console.log('ERROR CATCH ', e);
			process.exit(-1);
			
		}
		
		return response;
		
	}
	
	////**************end*****************////
	
	////***список категорий которые забираем с сайта***////
	
	const target = [ 
		'https://zakaz-sport.ru/catalog/velotrenazhery/',
		'https://zakaz-sport.ru/catalog/begovye_dorozhki/'  
	];
	
	////*****************end**************************////
	
	////***Начинаем парсинг***///
	
	let cc=0;
	
	
	const start = async (target) => {
		
		console.log('start')
		
		await client.access({
			************
		})
		
		
		let pageBegin = fs.readFileSync("page_gym.txt", "utf8");
		let targBegin = fs.readFileSync("target_gym.txt", "utf8");
		let firstCicle = true;
		
		console.log('остановились на разделе '+targBegin)
		
		
		for(targ of target){
			
			
			
			if(firstCicle){
				console.log('firstCicle', targ, targBegin, new Date());
				if(targ != targBegin.replace('\n','')){
					console.log('раздел прошли '+targ)
					continue; 
					}else{
					console.log('start_');
				}
			} 
			
			fs.writeFileSync("target_gym.txt", targ);
			
			cc++;
			var page = pageBegin; 
			
			let firstPage = await getPage(targ, 'firstPage');
			
			//console.log('firstPage',firstPage);
			
			for(url of firstPage.listUrls){
				
				// добавлять новые товары
				await getPage(url, 'getData'); 
				
			}
			
			for(let i = page; i<firstPage.count+1; i++){
				
				fs.writeFileSync("page_gym.txt", i);
				
				let nextPage = await getPage(firstPage.pageTemp+i, 'nextPage');
				
				for(url of nextPage.listUrls){
					
					// добавлять новые товары !!!!!!!!! именно эта строка
					await getPage(url, 'getData');
					
				}
				
			}
			
			firstCicle = false;
			pageBegin = 2;
			
		}
		
		fs.writeFileSync("page_gym.txt", 2);
		fs.writeFileSync("target_gym.txt", target[0]);
		
		
	}
	
	start(target);
	
	
	
	
	////**********end*********////
