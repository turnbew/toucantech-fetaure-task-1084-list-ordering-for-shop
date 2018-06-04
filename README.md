FOR PRIVACY AND CODE PROTECTING REASONS THIS IS A SIMPLIFIED VERSION OF CHANGES AND NEW FEATURES

TASK DATE: 24.11.2017 - Finished: 24.11.2017 

TASK LEVEL: (MEDIUM - I made it with ajax call)

TASK SHORT DESCRIPTION: 1084 (list ordering for shop)
	
GITHUB REPOSITORY CODE: feature/task-1084-list-ordering-for-shop

CHANGES
 
	IN FILES: 
		
		admin_products.php
	
			ADDED CODE: //inside index function 
			
				->append_js('module::move_table_rows.js')
				->append_css('module::move_table_rows.css')
	
			ADDED CODE 2: //A new function 
						
				/*
				 * Set order
				 *
				*/
				public function ajax_set_product_order()
				{ 
					if ( $this->input->is_ajax_request() ) 
					{
						$this->load->model('firesale/products_m');

						if ($this->products_m->set_products_order($this->input->post('id'), $this->input->post('direction'))) 
						{
							$this->pyrocache->delete_all('products_m', 'get_products_ids');	
							
							echo true;
						}
						else echo false;

						exit();
					}
				}				
		
		
		products_m.php
		
			ADDED CODE: //inside get_products_ids()
			
				->order_by('product_order', 'asc')
				
			ADDED CODE 2-3: //inside create and delete_product functions
			
				$this->set_product_orders();
				
			ADDED CODE 4: //some new functions 
			
					public function set_products_order($id, $direction)
					{
						$product_order = $this->get_product_order_by_id($id);
										
						//If direction is up, then the product's order will be decreased by one, so we need to replace the record with previous one
						//If direction is down, then the product's order will be increased by one, so we need to replace the record with next one
						$product = ($direction == 'up') 
										? $this->get_product_previous_in_order($product_order)
										: $this->get_product_next_in_order($product_order);
						..................

						return true;
					}
					
					
					public function get_product_order_by_id($id) 
					{
						$this->db->select('product_order');
						$this->db->from($this->_table);
						$this->db->where('id', $id);
						
						$records = $this->db->get()->result_array();
						
						return $records[0]['product_order'];
					}
					
					

					public function get_product_next_in_order($product_order) 
					{
						$this->db->from($this->_table);
						$this->db->order_by('product_order', 'ASC');
						$this->db->where('product_order >', $product_order);
						$this->db->limit(1);
						
						return $this->db->get()->result_array();
					}



					public function get_product_previous_in_order($product_order) 
					{
						$this->db->from($this->_table);
						$this->db->order_by('product_order', 'DESC');
						$this->db->where('product_order <', $product_order);
						$this->db->limit(1);

						return $this->db->get()->result_array();
					}	
					
					
					
					public function get_product_order_max() 
					{
						$this->db->from($this->_table);
						$this->db->order_by('product_order', 'DESC');
						$this->db->limit(1);
						
						$records = $this->db->get()->result_array();
						
						return (count($records) == 0) ? 0 : $records[0]['product_order'];
					}		



					public function set_product_orders() 
					{
						//Set the product_orders at every record
						$query = $this->db->query("Select id from " . $this->db->dbprefix($this->_table) . " ORDER BY product_order ASC, id ASC");
						$counter = 1;
						foreach ($query->result() as $row)
						{
							..............
						}
					}			
	
	
	
		table.php
		
			CHANGE CODE: 
			
				FROM: 
				
					 <tbody>
						<?php foreach ($products as $product): ?>
							<tr data-post-id="<?=$product['id']?>">
								
				TO: 
				
					<?php 
						$title_up = lang('firesale:move_product_up_title'); 
						$title_down = lang('firesale:move_product_down_title'); 		
						$counter = 1; 
						$l = count($products);
						foreach ($products as $product) : 
					?>
					<tr class="shop-product" data-row-counter="<?=$counter?>" id="product_<?=$product['id']?>">
						<td>
							<?php 
								$print = ($l == 1) 
											? 	'<span class="shop-product glyphicon glyphicon-chevron-up inactive"></span><br>' . 
												'<span class="shop-product glyphicon glyphicon-chevron-down inactive"></span>'
											:	'';
								$print = ($counter == 1) 
											? 	'<span class="shop-product glyphicon glyphicon-chevron-up inactive"></span><br>' . 
												'<span class="shop-product glyphicon glyphicon-chevron-down active" title="' . $title_down . '"></span>'
											:	$print;
								$print = ($counter == $l)
											?	'<span class="shop-product glyphicon glyphicon-chevron-up active" title="' . $title_up . '"></span><br>' . 
												'<span class="shop-product glyphicon glyphicon-chevron-down inactive"></span><br>'
											:	$print; 
								echo ($print == '') 
											?	'<span class="shop-product glyphicon glyphicon-chevron-up active" title="' . $title_up . '"></span><br>' . 
												'<span class="shop-product glyphicon glyphicon-chevron-down active" title="' . $title_down . '"></span>' 
											:	$print;  
								$counter++; 
							?>
						</td>					
	
	
	
		details.php
		
			ADDED CODE: //Inside install and upgrade function 
			
				$table = $this->db->dbprefix("firesale_products");

				//Add new coloumn to firesale_products table
				if (!$this->db->field_exists('product_order', $table)) 
				{
					$sql = 	"ALTER TABLE " . $table . " " . 
							"	ADD COLUMN product_order INT(11) NULL DEFAULT 0 " .
							"	AFTER tax_band";
					
					if ( ! $this->db->query($sql)) 
					{
						return false;
					}
				}
				
				//Set the product_orders at every record
				$query = $this->db->query("Select id from " . $table . " ORDER BY id ASC");
				$counter = 1;
				foreach ($query->result() as $row)
				{
					$this->db->set('product_order', $counter);
					$this->db->where('id', $row->id);
					$this->db->update($table); 					
					$counter++;
				}			
	
	
	
		network_settings.js
		
			ADDED CODE: 
			
				//Reorder shop product list clicking on order up/down arrows
				//MTR = short form of move_table_rows ... the file can be found assets/move-table-rows folder
				$(document).on('click', '.glyphicon-chevron-up.active', function() {
					MTR.setOrder('shop-product', $(this), 'up', 'network_settings/content/shop/products/ajax_set_product_order');
				});
				$(document).on('click', '.glyphicon-chevron-down.active', function() {
					MTR.setOrder('shop-product', $(this), 'down', 'network_settings/content/shop/products/ajax_set_product_order');
				});				
	
	
	
		firesale_lang.php
		
			ADDED CODE: 
			
				// Admin page Titles
				$lang['firesale:move_product_up_title']                 = 'Move product up in order';
				$lang['firesale:move_product_down_title']               = 'Move product down in order';				
	
		CHANGED QUITE NEW FILE: common_ajax.js
		
			CODE IN IT: 	
				
				/*
				 *	Callable variables
				 *
				 *	AJAX.asynch	- boolean, default value true - can be true or false 		 
				 *	AJAX.method 	- string, default value POST - can be GET, POST, PUT
				 *	AJAX.base_url	- string, default value = BASE_URI
				 *	AJAX.data	- string or object, default value empty object {} 	
				 *	AJAX.data_type	- string, default value text, can be text, xml, json, script or html
				 *	AJAX.success	- function, default: empty function
				 *	AJAX.error	- function, default: function(response) {console.log(response);alert('Something went wrong. Please, try again!');};
				 *	AJAX.complete	- function, default: empty function
				 *	AJAX.cache	- boolean, default: false - can be true or false
				 *	
				 *	
				 *	Callable functions
				 *	
				 * 	AJAX.construct(VOID) : VOID
				 *	AJAX.call(request_url, sent_params, callback_success_fn, callback_error_fn, callback_complete_fn)
				 *	
				*/

				
				(function($) {
						
					$.object_ajax = {
						
						//variables
						default_async				:	true,			//true or false - false is depricated, recommended true
						default_mehtod 				:	"",				//POST or GET
						default_base_url			:	"",		
						default_data				:	{},				//Can be anything, default is an empty object
						default_data_type			:	"",				//text, xml, json, script or html
						default_success_function	:	function(){},	//function will be evaulated if call was succes
						default_error_function		:	function(){},	//function will be evaulated if call caused error
						default_complete_function	:	function(){},	//function will be evaulated if call was completed
						default_cache				:	false,
						
						

						
						//functions
						/*
						 * @input: VOID
						 *
						 * @output: VOID
						*/
						__construct : function () 
						{
							AJAX.default_base_url = SITE_URL;			
							$.default_error_function = function(response) { 
															console.log(response);
															alert('Something went wrong. Please, try again!');
														};
							AJAX.default_method = "POST";
							AJAX.default_data_type = "text";			
						},	
						
						
						
						
						/*
						 * Makes a beautiful AJAX call using jQuery
						 *
						 * @input: (there's no Dependency Injection - params order important) 
						 *	- request_url: STRING, required : url to the server side script
						 *	- sent_params: ANY, non required, default value is {} : passed data to server side script
						 * 	- callback_success: FUNCTION non required, default is empty function : callback function which will be evaulated when ajax call was success
						 * 	- callback_error: FUNCTION non required, default is empty function : callback function which will be evaulated when ajax call produced error
						 * 	- callback_complete: FUNCTION non required, default is empty function : callback function which will be evaulated when ajax call was completed
						 *
						 * @output: VOID or returns response (if call is synchronous - not recommended)
						*/
						call : function(request_url, sent_params, callback_success, callback_error, callback_complete)
						{
							var data 		= (sent_params != undefined) 		? sent_params 		: AJAX.default_data;
							var fn_succes 	= (callback_success != undefined) 	? callback_success 	: AJAX.default_success_function;
							var fn_error 	= (callback_error != undefined) 	? callback_error 	: AJAX.default_error_function;
							var fn_complete = (callback_complete != undefined)	? callback_complete	: AJAX.default_complete_function;
							var response 	= $.ajax({
												url 	: AJAX.default_base_url + request_url, 
												type 	: AJAX.default_method,
												data 	: data,			
												dataType: AJAX.default_data_type,
												async	: true,
												success	: fn_succes, 
												error	: fn_error,
												complete: fn_complete
											});		
							if (!AJAX.default_async) return $.trim(response);
						},
						
						
						
					}; //END $.object_ajax object

					
					//Global variables
					AJAX = 	$.object_ajax;
					
					
				})(jQuery);


				//when DOM is loaded
				$(function() {
					
					//Initialization
					AJAX.__construct();
				})			


	
		ADDED NEW FILE: \network-site\addons\default\modules\network_settings\css\move_table_rows.cssű
		
			CODE IN IT: 
			
				span.glyphicon-chevron-up.inactive,
				span.glyphicon-chevron-down.inactive,
				.news-cat-order-arrows > span {
					color: #ccc;
					cursor: auto;
					background-color: rgba(255,255,255,0); /*background color and opacity together*/
				}
				span.glyphicon-chevron-up.active,
				span.glyphicon-chevron-down.active {
					color: #428bca;
					cursor: pointer;
					background-color: rgba(255,255,255,0); /*background color and opacity together*/
				}
				span.glyphicon-chevron-up.active:hover,
				span.glyphicon-chevron-down.active:hover {
					color: #2a6496;
				}
				.shop-product {
					background-color: white;
					transition: background-color 1s;
				}
				.shop-product.flash-background{
					background-color: #a7e886;
					transition: background-color 1s;
				}
					
					
					
		ADDED NEW FILE: \network-site\addons\default\modules\network_settings\js\move_table_rows.js
		
			CODE IN IT: 
			
					/*
					 *	Callable variables
					 *
					 *	AJAX.row_animation	- css class which will be set after the table rows were replaced - note - table row can not be animated, if you'd like then need to be wrapped into div		 
					 *	AJAX.set_timeout 	- css class will be deleted after this time
					 *	
					 *	
					 *	Callable functions
					 *	
					 * 	AJAX.setOrder(list_name, arrow_element, direction, request_url)
					 * 						@param list_name: name of the list, for example: shop-product, generally css selector of tr_element
					 * 						@param arrow_element: dom object, generally the up and down arrows 
					 * 						@param direction: can be up or down 
					 * 						@param request_url:	path to the ajax function 		 
					 *	
					*/
					
				
				
				(function($) {
						
					$.move_table_rows = {
							
						//Variables
						row_animation 		: "flash-background",
						set_timeout 		: 500,
						
						//Methods
						setOrder : function(list_name, arrow_element, direction, request_url, console) 
						{
							var show_response = (console != undefined) ? console : '';
							var tr_element = arrow_element.parent().parent(); 
							var data = {
									'id' : tr_element.attr('id').replace('product_', ''), 
									'direction' : direction
								};
							AJAX.call(request_url, data, function(response) {
								if (response) {
									var row_counter = tr_element.attr("data-row-counter");
									(direction == 'up') 
											? MTR._moveUp(list_name, arrow_element, row_counter)
											: MTR._moveDown(list_name, arrow_element, row_counter);
								}
								if (show_response == 'console') {
									alert(response);
								}
								else if (show_response == 'alert') {
									alert(response);
								}
							});
							
							return true; 
						},	

						
						
						_moveUp : function(list_name, arrow_element, row_counter) 
						{
							var row = arrow_element.parent().parents("tr:first");			
							var rows = arrow_element.parent().parent().parent().find("tr." + list_name);	

							row.insertBefore(row.prev());
							MTR._setNewOrder(list_name, row_counter, 'up');
							MTR._animationAfterMoving(row, MTR.row_animation, MTR.set_timeout);
							
							if (row_counter == rows.length) {
								MTR._setArrowsClass(list_name, rows.length, 'active', 'inactive');
								MTR._setArrowsClass(list_name, parseInt(row_counter) - 1, 'active', 'active');
							}
							else if (row_counter == 2) {
								MTR._setArrowsClass(list_name, parseInt(row_counter) - 1, 'inactive', 'active');
								MTR._setArrowsClass(list_name, row_counter, 'active', 'active');
							}
						},
						

						
						_moveDown : function(list_name, arrow_element, row_counter) 
						{
							var row = arrow_element.parent().parents("tr:first");
							var rows = arrow_element.parent().parent().parent().find("tr." + list_name);	
								
							row.insertAfter(row.next());
							MTR._setNewOrder(list_name, row_counter, 'down');
							MTR._animationAfterMoving(row, MTR.row_animation, MTR.set_timeout);
							
							if (row_counter == 1) {
								MTR._setArrowsClass(list_name, row_counter, 'inactive', 'active');
								MTR._setArrowsClass(list_name, parseInt(row_counter) + 1, 'active', 'active');
							}
							else if (row_counter == rows.length - 1) {
								MTR._setArrowsClass(list_name, rows.length, 'active', 'inactive');
								MTR._setArrowsClass(list_name, row_counter, 'active', 'active');
							}
						},
						
						
						
						
						_setNewOrder : function(list_name, row_counter, direction) 
						{
							var new_row_counter = (direction == 'up') ? parseInt(row_counter) - 1 : parseInt(row_counter) + 1;
							var element1 = $('.' + list_name + '[data-row-counter="' + row_counter + '"]');
							var element2 = $('.' + list_name + '[data-row-counter="' + new_row_counter + '"]');
							
							element1.attr('data-row-counter', new_row_counter);
							element2.attr('data-row-counter', row_counter);
						},

						
						
						_animationAfterMoving : function(dom_element, animation_css_class, speed) 
						{
							dom_element.addClass(animation_css_class);
							setTimeout(function() {
								dom_element.removeClass(animation_css_class);
							}, speed);
						},	
						
						
						
						_setArrowsClass : function(list_name, row_counter, up_addclass, down_addclass)		
						{
							var arrow_up = $('.' + list_name + '[data-row-counter="' + row_counter + '"]').find('span.glyphicon-chevron-up');
							var arrow_down = $('.' + list_name + '[data-row-counter="' + row_counter + '"]').find('span.glyphicon-chevron-down');

							arrow_up.removeClass('inactive').removeClass('active').addClass(up_addclass);
							arrow_down.removeClass('inactive').removeClass('active').addClass(down_addclass);
						},					
						
					}; //END $.move_table_rows object

					
					//GLOBAL variables of $.move_table_rows object
					MTR		 			= $.move_table_rows;
					
				})(jQuery);
				
