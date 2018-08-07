# CONFIGURACION DE ADMINISTRACIÓN DE USARIOS DE LDAP Y JBPM DESDE APACHE DIRECTORY STUDIO

La instalación descrita es para tener interconectado el LDAP con el JBPM para ello es necesario tener configurado LDAP con postgres y JBPM con postgres, estas instalaciones se las puede ver en el pdf llamado "manual_de_configuracion.pdf".

Entonces una vez realizado los pasos del manual de configuración es necesario manipular la base de datos del LDAP, aquí es importante indicar que se pueden crear funciones, triggers, etc sin restricciones tomando en cuenta que el programa que conectemos para visualizacion como puede ser Apache Directory Studio, JExplorer, etc., haran las llamadas que se encuentran en la base de datos del LDAP en la tabla ldap_attr_mappings la cual fue creada con la metadata del ldap.

Como la conexion con el JBPM depende de los archivos "users.properties" y "roles.properties" ubicados en /home/rfam/jbpm/jbpm-installer/wildfly-8.1.0.Final/standalone/configuration/users.properties en nuestro caso, es necesario hacer que los usuarios que se inserten en LDAP esten interconectados con estos archivos a travez del postgres. Para ello hay que dar permisos para modificación de los archivos por esta razon desde el terminal se ejecuta 

	chmod -R 777 /home/rfam/jbpm/jbpm-installer/wildfly-8.1.0.Final/standalone/configuration

por otra parte creamos dos funciones que modifiquen los archivos, la que modifica "users.properties":

	-- FUNCTION: public.print_mail_password()

	-- DROP FUNCTION public.print_mail_password();

	CREATE OR REPLACE FUNCTION public.print_mail_password(
		)
  	  RETURNS integer
  	  LANGUAGE 'sql'

  	  COST 100
  	  VOLATILE 
	AS $BODY$

   	 copy (select concat_ws('=',mail,password) from mails, persons where mails.pers_id = persons.id) TO '/home/rfam/jbpm/jbpm-installer/wildfly-8.1.0.Final/standalone/configuration/users.properties';
   	 select max(id) from persons;

	$BODY$;

	ALTER FUNCTION public.print_mail_password()
   	 OWNER TO postgres;

y la que modifica "roles.properties": 

	-- FUNCTION: public.add_description(character varying, integer)

	-- DROP FUNCTION public.add_description(character varying, integer);

	CREATE OR REPLACE FUNCTION public.add_description(
		character varying,
		integer)
   	 RETURNS integer
   	 LANGUAGE 'sql'

   	 COST 100
   	 VOLATILE 
	
	AS $BODY$
	select setval ('descriptions_id_seq', (select case when max(id) is null then 1 else max(id) end from descriptions));
	insert into descriptions (id,description,pers_id) 
    	values (nextval('descriptions_id_seq'),$1,$2);
    	copy public.descriptions (description) TO '/home/rfam/jbpm/jbpm-installer/wildfly-8.1.0.Final/standalone/configuration/roles.properties';    	
    	select max(id) from descriptions;

	$BODY$;

	ALTER FUNCTION public.add_description(character varying, integer)
   	 OWNER TO postgres;

Estas funciones la podemos ejecutar con triggers en el momento de hacer inserciones, updates, etc., o como en nuestro caso cada vez que se hace una insercion o modificacion del password y descripcion para ello en el caso de la modificacion del password se hizo que se ejecute la funcion cuando el ldap inserta el mismo a travez de la tabla ldap_attr_mappings de la siguiente manera:

	update ldap_attr_mappings set add_proc =
	(select  $aesc6$UPDATE persons SET password=? WHERE id=?; select print_mail_password()$aesc6$)
	where id=5

mientras que en el caso de la descripción se realizo esta ejecución cuando se utiliza la funcion add_description agregandole la linea "copy public.descriptions (description) TO '/home/rfam/jbpm/jbpm-installer/wildfly-8.1.0.Final/standalone/configuration/roles.properties';" que dando asi:

	-- FUNCTION: public.add_description(character varying, integer)

	-- DROP FUNCTION public.add_description(character varying, integer);

	CREATE OR REPLACE FUNCTION public.add_description(
		character varying,
		integer)
   	 RETURNS integer
   	 LANGUAGE 'sql'

  	  COST 100
  	  VOLATILE 
   	 ROWS 0
	AS $BODY$

	select setval ('descriptions_id_seq', (select case when max(id) is null then 1 else max(id) end from descriptions));
	insert into descriptions (id,description,pers_id) 
    	values (nextval('descriptions_id_seq'),$1,$2);
    	copy public.descriptions (description) TO '/home/rfam/jbpm/jbpm-installer/wildfly-8.1.0.Final/standalone/configuration/roles.properties';    	
    	select max(id) from descriptions;

	$BODY$;

	ALTER FUNCTION public.add_description(character varying, integer)
    	OWNER TO postgres;

Por tanto ya se podria administrar los usuarios del JBPM desde Apache Directory Studio, JEXPLORER, etc. hay que tomar en cuenta que las funciones se ejecutan cuando corro estos programas mas no cuando hago cambios en el postgres sin embargo todos los cambios que se hagan en el postgres se veran en el Apache Directory Studio, JEXPLORER, etc. y se puede hacer que se grabe en los documentos deseados desde el postgres con:

	copy public.descriptions (description) TO '/home/rfam/jbpm/jbpm-installer/wildfly-8.1.0.Final/standalone/configuration/users.properties'; 
	copy public.descriptions (description) TO '/home/rfam/jbpm/jbpm-installer/wildfly-8.1.0.Final/standalone/configuration/roles.properties'; 

aquí se prefirio no hacer un trigger para no estar grabando a cada rato que hagamos cambios directo en el postgres si no una sola vez cuando se hagan terminado de hacer los cambios deseados directamente en la base de datos con la finalidad de minimizar el uso de recursos.

Nota: Lo que se esta haciendo es copiar los atributos de las tablas que necesitamos en los archivos deseados pero todos los atrbutos, por lo que para optimizar seria necesario buscar el comando de postgres que permita modicar el archivo (no reemplazarlo como sucede ahora) con la finalidad de que cada nuevo dato que se inserte se vaya colocando, sin embargo abria que tener encuenta borrados de datos, triggers que se deben crear, etc.
