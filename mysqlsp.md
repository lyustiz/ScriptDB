CREATE DEFINER=`root`@`localhost` PROCEDURE `psCrearCartones`(IN `parIdProgramacionJuego` int)
BEGIN

-- FECHA      -- AUTOR  -- CAMBIOS
-- 03/01/2024 -- YUSTIZ -- AJUSTE DE CONTEO DE HOJAS/LINEAS PARA JUEGOS CON CARTON INICIAL > 1

DECLARE varActualizado TIMESTAMP; 

BEGIN
	GET DIAGNOSTICS CONDITION 1 @sqlstate = RETURNED_SQLSTATE, @errno = MYSQL_ERRNO, @text = MESSAGE_TEXT;
END;


  --  datos de cartones a insertar
	SELECT pj.CartonInicial,
		     pj.CartonFinal,
		     PJ.IdSerie,
				 lt.Tabla,
				 pj.CartonesAleatorios,
				 pj.CartonesPorPagina,
				 pj.CartonesPorHoja,
				 pj.CartonesPorLinea
	  INTO @CartonInicial,
		     @CartonFinal,
		     @IdSerie,
				 @TablaSerie,
				 @CartonesAleatorios,
				 @CartonesXPagina,
				 @CartonesXHoja,
				 @CartonesXLinea
    FROM programacionjuegos pj 
		JOIN listablas lt ON lt.IdLisTablas = pj.IdSerie
   WHERE pj.IdProgramacionJuego = parIdProgramacionJuego;

	 
	 IF IFNULL(@CartonesXPagina, 0) < 1 OR IFNULL(@CartonesXHoja, 0) < 1 OR IFNULL(@CartonesXLinea, 0) < 1 THEN
    SIGNAL SQLSTATE '45000'	SET MESSAGE_TEXT = 'Error en configuracion de Juego PAG/HOJ/LIN'; 
  END IF;
	
	
	IF IFNULL(@CartonInicial, 0) < 1 THEN
    SIGNAL SQLSTATE '45000'	SET MESSAGE_TEXT = 'El Juego no existe'; 
  END IF;
	
	SET @Numerohoja := 1;

	--  buscar cartones existentes y modificar rangos (cambio de rangos)
	SELECT IFNULL(MAX(Numero) + 1, @CartonInicial), 
	       IFNULL(MAX(NumeroHoja) + 1, @Numerohoja)
	  INTO @CartonInicial, 
		     @Numerohoja    
    FROM cartones 
	 WHERE IdProgramacionJuego = parIdProgramacionJuego;
	 
	 SET @HojaInicial = @Numerohoja; 
	 
  IF(@CartonFinal > @CartonInicial ) THEN
	   SET @sqlstr :=	CONCAT('INSERT INTO cartones (IdProgramacionJuego,
	                                              IdSerie,
																								Numero,
	                                              NumeroHoja,
																								Estado) ',
	       'SELECT ', parIdProgramacionJuego, ',' 
				          , @IdSerie , ','
									, 'Numero' , ','
	                , '@Numerohoja := IF( (Numero > ', @CartonInicial ,') 
									                          AND ( (Numero - ', @CartonInicial ,' ) % ', @CartonesXHoja, ' = 0), 
								                       @Numerohoja + 1, @Numerohoja
																		  ) as NroHoja ', ','
									, '''A''', ' ',
	         'FROM ', @TablaSerie, ' ',
	        'WHERE Numero BETWEEN ', @CartonInicial ,' AND ', @CartonFinal);		

		PREPARE stmt FROM @sqlstr;
		EXECUTE stmt;
		DEALLOCATE PREPARE stmt;
		
	  -- todo lleva numero de linea
		-- SET @NumeroAux := @CartonInicial - IF(@CartonInicial > 1, 2, 1);
		-- ET @NumeroAux := @CartonInicial -  IF( @HojaInicial > 1, 1, IF(@CartonInicial > 1, 2, 1))  ;
		SET @Numerohoja = @HojaInicial ;	
		
		-- NUMERO DE  LINEA
		SET @NumeroLinea := 0;
		SET @CantLineas := round(@CartonesXHoja / @CartonesXLinea, 0);

		IF (IFNULL(@CartonesAleatorios, 'N') = 'S') THEN
		
			SET @NumeroAux = -1;  -- FIX ERROR CONTEO EN CREAR CARTONES INICIAL > 1
					
				UPDATE cartones c1
			INNER JOIN (
				  SELECT @NumeroAux := @NumeroAux + 1 AS AUXILIAR,
			           @NumeroAux % @CartonesXHoja AS RESTO,
							   numero,  
							   @NumeroHoja := IF( ((@NumeroAux > 0) AND (@NumeroAux % @CartonesXHoja = 0 )),
							                      @NumeroHoja + 1,  @NumeroHoja) AS Nrohoja, 
							   @NumeroLinea := IF( (@NumeroLinea >= @CantLineas),  1,  (@NumeroLinea + 1) ) AS Nrolinea
				   FROM (  
								  SELECT Numero 
									  FROM cartones 
									 WHERE IdProgramacionJuego = parIdProgramacionJuego
								     AND 	Numero BETWEEN @CartonInicial AND @CartonFinal
							  ORDER BY RAND()
							) c
			) c2 ON c1.Numero = c2.numero AND c1.IdProgramacionJuego = parIdProgramacionJuego
			SET c1.NumeroHoja = c2.Nrohoja, c1.NumeroLinea = c2.Nrolinea;
			

	 ELSE
			
			SET @repetir = 0;
			SET @NumeroLinea = 1;
			
			select 2, @CartonInicial, @CartonFinal, @CartonesXHoja, @CartonesXLinea, @NumeroAux, @NumeroHoja, @NumeroLinea, @repetir, @CantLineas;
			
			UPDATE cartones c1
			INNER JOIN (
				SELECT Numero numeroCarton,
               @NumeroLinea := IF(@NumeroLinea >= @CantLineas and @repetir >= @CartonesXLinea, 
																 1,  
																 IF(@repetir >= @CartonesXLinea , @NumeroLinea + 1 , @NumeroLinea)
															 ) AS NumeroLinea,
							 @repetir:= IF(@repetir >= @CartonesXLinea, 1 ,   @repetir+1) AS RepetirLinea,
							 Numero,  
							 @NumeroHoja := IF( (Numero > @CartonInicial) AND ((Numero-1) % @CartonesXHoja = 0),@NumeroHoja + 1 , @NumeroHoja) AS Nrohoja
							
				 FROM (  
								 SELECT Numero 
									 FROM cartones 
									WHERE IdProgramacionJuego = parIdProgramacionJuego
									AND 	Numero BETWEEN @CartonInicial AND @CartonFinal
							) c
				) c2 ON c1.Numero = c2.numero AND c1.IdProgramacionJuego = parIdProgramacionJuego
			  SET c1.NumeroLinea = c2.NumeroLinea;
			
	 END IF;
		
	  --  Asignacion automatica de cartones
		SELECT pg.Valor  
			INTO @AsignacionAutomatica
			FROM parametrosgenerales pg
		WHERE pg.Referencia = 'distribucionautomaticahojas';
	
		IF IFNULL(@AsignacionAutomatica,2) =  1 THEN
		   CALL psAutoAsignarHojasVendedores(parIdProgramacionJuego);	
		END IF;
    
	END IF;

END

CREATE DEFINER=`root`@`localhost` PROCEDURE `psAsignarCartonesVendedor`(IN `parIdProgramacionJuego` int, IN `parIdVendedor` int)
BEGIN

DECLARE varActualizado TIMESTAMP;
DECLARE varCartonesPagina INT;

BEGIN
	GET DIAGNOSTICS CONDITION 1 @sqlstate = RETURNED_SQLSTATE, @errno = MYSQL_ERRNO, @text = MESSAGE_TEXT;
	-- SET @errormsj = CONCAT("0|ERROR ", @errno, " (", @sqlstate, "): ", @text);
  -- SELECT @errormsj;
END;

  SET varActualizado:= CURRENT_TIMESTAMP();
	
	-- cantidad de cartones x pagina segun parametros
	SELECT CartonesPorPagina INTO varCartonesPagina
	  FROM programacionjuegos 
	 WHERE IdProgramacionJuego = parIdProgramacionJuego;

	-- paginas entregadas en el juego a todos los vendedores exeptuando el vendodor a asignar
	SELECT IFNULL(MAX(NumeroPagina),0) INTO @NumeroPaginasJuego
	  FROM cartones 
	 WHERE IdProgramacionJuego = parIdProgramacionJuego
	   AND Idvendedor IS NOT NULL;

  -- paginas que ya posee el vendedor
	SELECT MAX(NumeroPagina) INTO @NumeroPaginaVendedor 
	  FROM cartones 
	 WHERE IdVendedor = parIdVendedor 
	   AND IdProgramacionJuego = parIdProgramacionJuego;

	IF @NumeroPaginaVendedor IS NULL THEN 
		 -- verificar paginas de otros vendedores
	   SET @NumeroPaginaVendedor = IF( @NumeroPaginasJuego = 0, 1, @NumeroPaginasJuego + 1 );
	   SET @CartonesXPagina := 0;
	ELSE
		-- verificar la ultima pagina del vendedor y si ya ha sido impresa
		 SELECT COUNT(Numero), SUM(Impreso) INTO @CartonesXPagina, @PoseeCartonesImpresos
			 FROM cartones 
			WHERE IdVendedor = parIdVendedor 
				AND IdProgramacionJuego = parIdProgramacionJuego
				AND NumeroPagina = @NumeroPaginaVendedor;

		IF @PoseeCartonesImpresos > 0 THEN
		   -- la pagina ya ha sido impresa selecciono la siguiente
			IF (@NumeroPaginaVendedor + 1) <= @NumeroPaginasJuego THEN
			   SET @NumeroPaginaVendedor :=  @NumeroPaginasJuego + 1; 
			ELSE
			   SET @NumeroPaginaVendedor :=  @NumeroPaginaVendedor + 1; 
			END IF;
			-- reinicio el contador de cartones a 0
			SET @CartonesXPagina := 0;
		END IF;
			
  END IF;

  -- asignos cartones segun las hojas que no han sido asignados anteriormente y los agrupo por paginas
	UPDATE cartones c1
		INNER JOIN (
									SELECT IdCarton, 
												 @NumeroPaginaVendedor := IF( @CartonesXPagina >= varCartonesPagina, IF( @NumeroPaginaVendedor > @NumeroPaginasJuego, @NumeroPaginaVendedor + 1, @NumeroPaginasJuego +1 ), @NumeroPaginaVendedor) AS Nropagina,
												 @CartonesXPagina := IF( @CartonesXPagina >= varCartonesPagina, 1, @CartonesXPagina + 1)  AS CartonesXpagina
										FROM cartones 
									 WHERE idvendedor IS NULL
										 AND IdProgramacionJuego = parIdProgramacionJuego
										 AND numeroHoja IN (SELECT vh.NumeroHoja
																					FROM vendedoreshojas vh
																				 WHERE vh.IdProgramacionJuego = parIdProgramacionJuego
																					 AND vh.IdVendedor =  parIdVendedor
																					 AND vh.Estado = 'A') 
									  ORDER BY NumeroHoja
		) c2 ON c1.IdCarton = c2.IdCarton
		SET c1.IdVendedor = parIdVendedor,
				c1.NumeroPagina = c2.Nropagina,
				c1.Actualizado = varActualizado;

END






CREATE DEFINER=`root`@`localhost` PROCEDURE `psTrasladarJuego`(IN parIdProgramacionJuego INT, 
IN parIdVendedorOrigen INT, 
IN parIdvendedorDestino INT, 

IN parIdUsuario INT)
BEGIN

DECLARE varActualizado              TIMESTAMP; 
DECLARE varIdVendedorResumenDestino INT DEFAULT 0;
DECLARE varIdVendedorResumenOrigen  INT DEFAULT 0;
DECLARE varValorcarton        DOUBLE DEFAULT 0;
DECLARE varEstaSincronizado   INT DEFAULT 0;
DECLARE varEstaVendido        INT DEFAULT 0;
DECLARE varCantidadCartones   INT DEFAULT 0;
DECLARE varValorTotal         DOUBLE DEFAULT 0;
DECLARE varTotalCartonesDest  INT DEFAULT 0;
DECLARE varTotalCartonesDestFinal  INT DEFAULT 0;
DECLARE varValorTotalDest     DOUBLE DEFAULT 0;
DECLARE varValorPendienteDest DOUBLE DEFAULT 0;

DECLARE varRegistroAnterior   VARCHAR(300);
DECLARE varRegistroNuevo      VARCHAR(300);

DECLARE EXIT HANDLER FOR SQLEXCEPTION
  BEGIN
	  GET DIAGNOSTICS CONDITION 1 @sqlst = RETURNED_SQLSTATE, @errno = MYSQL_ERRNO, @texterror = MESSAGE_TEXT;
    ROLLBACK;
		SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = @texterror;
END;

  -- valor carton
  SELECT pj.ValorCarton, CartonesPromocion
	  INTO varValorcarton, @CartonesPromocion
	  FROM programacionjuegos pj
	 WHERE pj.IdProgramacionJuego = parIdProgramacionJuego;	

	-- busco la paginas del vendedor origen
	SELECT SUM(SinconizadoApp), SUM(IF(Estado = 'V', 1, 0)), IFNULL(COUNT(IdCarton), 0)
	  INTO varEstaSincronizado, varEstaVendido, varCantidadCartones
		FROM cartones 
	 WHERE IdVendedor = parIdVendedorOrigen 
	   AND IdProgramacionJuego = parIdProgramacionJuego;

	IF varCantidadCartones = 0 THEN
		SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'El vendedor no posee cartones ';
	END IF;

	IF varEstaSincronizado > 0 THEN
		SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Posee cartones sincrnizados con la APP '; 
	END IF;

	IF varEstaVendido > 0 THEN
		SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Posee cartones vendidos ';
	END IF;

	START TRANSACTION;

	-- *** trasladar hojas entre vendedores ***
	-- copio las hojas del vendedor origen hacia vendedor destino 

	INSERT INTO vendedoreshojas(IdProgramacionJuego, IdVendedor, NumeroHoja, Estado)
       SELECT IdProgramacionJuego, parIdvendedorDestino, NumeroHoja, Estado 
		     FROM vendedoreshojas
		    WHERE IdVendedor = parIdVendedorOrigen
		      AND IdProgramacionJuego = parIdProgramacionJuego 
			    AND Estado = 'A';

	-- inactivo las hojas del vendedor origen 
	UPDATE vendedoreshojas 
	   SET Estado = 'I'
	 WHERE IdVendedor = parIdVendedorOrigen
	   AND IdProgramacionJuego = parIdProgramacionJuego 
	   AND Estado = 'A';
  
   -- *** actualizar resumenes ***

	-- verifico existencia del resumen de vendedor destino 
  SELECT IFNULL(IdVendedorResumen, 0), IFNULL(TotalCartones, 0), ValorTotal, ValorPendiente
	  INTO varIdVendedorResumenDestino, varTotalCartonesDest, varValorTotalDest, varValorPendienteDest
	  FROM vendedoresresumen
   WHERE IdProgramacionJuego = parIdProgramacionJuego
	   AND IdVendedor = parIdvendedorDestino; -- FIX  ERRROR EN EXISTENCIA DE VARIOS RESUMENNES

	SET varValorTotal := ( varCantidadCartones / @CartonesPromocion ) * varValorcarton;

	 -- creo o actualizo el resumen del vendedor destino
	IF varIdVendedorResumenDestino = 0 THEN 

		INSERT INTO vendedoresresumen(IdProgramacionJuego, 
										              IdVendedor, 
										              TotalCartones, 
										              ValorTotal, 
										              ValorPendiente, 
										              Estado)
								           VALUES(parIdProgramacionJuego, 
										              parIdvendedorDestino, 
										              varCantidadCartones, 
										              (varCantidadCartones / @CartonesPromocion) * varValorcarton, 
										              (varCantidadCartones / @CartonesPromocion) * varValorcarton, 
										              'A');

		SELECT LAST_INSERT_ID() INTO varIdVendedorResumenDestino;

		-- *** guardo auditoria ***
		INSERT INTO auditoria(IdUsuario, 
													Tabla, 
													Accion, 
													Complemento,
													IdProgramacionJuego,
													IdVendedorOrigen,
													IdVendedorDestino,
													RegistroAnterior,
													RegistroNuevo
													)
									 VALUES(parIdUsuario,
													'vendedoresresumen', 
													'INS',
													'TRASLADAR JUEGO', 
													parIdProgramacionJuego,
													parIdVendedorOrigen,
													parIdvendedorDestino,
													'',
													CONCAT('IdVendedorResumen:', varIdVendedorResumenDestino, '|TotalCartones:',varCantidadCartones)

													); 

	ELSE

		SET varTotalCartonesDestFinal := IFNULL(varTotalCartonesDest, 0) + varCantidadCartones;

			UPDATE vendedoresresumen 
			   SET TotalCartones  = varTotalCartonesDestFinal,
			       ValorTotal     = varTotalCartonesDestFinal + varValorTotal,
			       ValorPendiente = varTotalCartonesDestFinal + varValorTotal,
				     Estado         = 'A'
		   WHERE IdVendedorResumen = varIdVendedorResumenDestino;

	 	-- *** guardo auditoria ***
		INSERT INTO auditoria(IdUsuario, 
	                      Tabla, 
						            Accion, 
												Complemento,
												IdProgramacionJuego,
												IdVendedorOrigen,
												IdVendedorDestino,
												RegistroAnterior, 
						            RegistroNuevo
												)
                 VALUES(parIdUsuario, 
						            'vendedoresresumen', 
												'UPD',
						            'TRASLADAR JUEGO', 
												parIdProgramacionJuego,
												parIdVendedorOrigen,
												parIdvendedorDestino,
						            CONCAT('IdVendedorResumen:', varIdVendedorResumenDestino, '|TotalCartones:',varTotalCartonesDest ), 
						            CONCAT('IdVendedorResumen:', varIdVendedorResumenDestino, '|TotalCartones:',varTotalCartonesDestFinal)
												); 

	END IF;

	-- ELIMINO  resumen vendedo destino
  DELETE FROM vendedoresresumen 
	       WHERE IdProgramacionJuego = parIdProgramacionJuego
				 AND IdVendedor = parIdVendedorOrigen;

  -- *** guardo auditoria ***
		INSERT INTO auditoria(IdUsuario, 
	                      Tabla, 
						            Accion, 
												Complemento,
												IdProgramacionJuego,
												IdVendedorOrigen,
												IdVendedorDestino,
												RegistroAnterior,
											  RegistroNuevo	
												)
                 VALUES(parIdUsuario, 
						            'vendedoresresumen', 
												'DEL',
						            'TRASLADAR JUEGO', 
												parIdProgramacionJuego,
												parIdVendedorOrigen,
												parIdvendedorDestino,
						            CONCAT('IdVendedorResumen:', varIdVendedorResumenOrigen, '|TotalCartones:',varCantidadCartones ),
											  ''	
												);

	-- *** actualizo tabla cartones ***
	UPDATE cartones 
	   SET IdVendedor  = parIdvendedorDestino,
		     Impreso     = 0, -- quito la marca de impreso
		     Actualizado = varActualizado
	 WHERE IdVendedor  = parIdVendedorOrigen 
	   AND IdProgramacionJuego = parIdProgramacionJuego;

	COMMIT;

END


CREATE DEFINER=`soporte`@`%` PROCEDURE `cha_psEncontrarGanadores`(IN parIdSorteo int)
BEGIN

 DECLARE varActualizado TIMESTAMP; 
 DECLARE varDone INT DEFAULT 0;
 DECLARE varIdConfiguracionSorteo INT DEFAULT 0;
 DECLARE varIdPremioSorteo INT;
 DECLARE varIdPremio INT;
 DECLARE varNombrePremio VARCHAR(80);
 DECLARE varMultiplicador INT; 
 DECLARE varValorPremio INT; 
 DECLARE varIsBalota1 INT DEFAULT 0; 
 DECLARE varIsBalota2 INT DEFAULT 0; 
 DECLARE varIsBalota3 INT DEFAULT 0; 
 DECLARE varIsBalota4 INT DEFAULT 0; 
 DECLARE varIsBalota5 INT DEFAULT 0; 
 DECLARE varIsBalota6 INT DEFAULT 0; 
 DECLARE varOrdenEstricto CHAR(1) DEFAULT 0; 
 DECLARE varIsExtra INT;

 DECLARE curPremios CURSOR FOR (SELECT prs.IdPremioSorteo,
                                       prs.IdPremio,
                                       pre.Nombre,
                                       prs.Multiplicador, 
																			 prs.ValorPremio, 
																			 pre.Balota1,	
																			 pre.Balota2,	
																			 pre.Balota3,	
																			 pre.Balota4,	
																			 pre.Balota5,	
																			 pre.Balota6, 
																			 pre.OrdenEstricto,	
																			 pre.Extra
																  FROM cha_premiossorteo prs
																  JOIN cha_premios pre ON pre.IdPremio = prs.IdPremio
																 WHERE prs.IdConfiguracionSorteo = varIdConfiguracionSorteo
																 ORDER BY prs.Multiplicador DESC);														
												  																
 DECLARE CONTINUE HANDLER FOR NOT FOUND SET varDone = 1;

 DECLARE EXIT HANDLER FOR SQLEXCEPTION
 BEGIN
		GET DIAGNOSTICS CONDITION 1 @sqlst = RETURNED_SQLSTATE, @errno = MYSQL_ERRNO, @texterror = MESSAGE_TEXT;
		-- SET @texterror = SUBSTR(IFNULL(@texterror, 'desconocido'),1,120);
		ROLLBACK;
		SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = @texterror;
 END;

  -- Datos Sorteo
	SELECT IdLoteria,
				 IdConfiguracionSorteo,
				 Balota1,	
				 Balota2,	
				 Balota3,	
				 Balota4,	
				 Balota5,	
				 Balota6,	
				 Extra,
				 Estado
		INTO @IdLoteria,
				 varIdConfiguracionSorteo,
				 @sorBalota1,	
				 @sorBalota2,	
				 @sorBalota3,	
				 @sorBalota4,	
				 @sorBalota5,	
				 @sorBalota6,	
				 @sorExtra,
				 @Estado
		 FROM cha_sorteos 
	  WHERE IdSorteo = parIdSorteo;

   IF IFNULL(varIdConfiguracionSorteo,0) < 1 THEN
    SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Sorteo no valido';
   END IF; 
	 
	 IF @Estado = 'P' THEN
    SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Debe asignar Numeros Ganadores al Juego ';
   END IF;
	 
	 IF @sorBalota1 IS NULL OR @sorBalota2 IS NULL OR @sorBalota3 IS NULL OR @sorBalota4 IS NULL THEN
    SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Asignacion de numeros Ganadores Invalida';
   END IF;
   
	START TRANSACTION;
  -- --------------------------------------------------
		
	OPEN curPremios;
	loopQuery: LOOP
  FETCH curPremios INTO varIdPremioSorteo, varIdPremio, varNombrePremio, varMultiplicador, varValorPremio, 
												varIsBalota1, varIsBalota2, varIsBalota3, varIsBalota4, varIsBalota5, varIsBalota6, 
												varOrdenEstricto, varIsExtra;
														
     IF varDone = 1 THEN
			  LEAVE loopQuery;
		 END IF;
			
			IF varOrdenEstricto = 'S' THEN -- DIRECTO
			
			 /*SELECT 'DIRECTO', varNombrePremio, varIsBalota1, varIsBalota2, varIsBalota3, varIsBalota4, varIsBalota5, varIsBalota6, 
												varOrdenEstricto, varIsExtra,
												@sorBalota1, @sorBalota2, @sorBalota3,@sorBalota4,@sorBalota5,@sorBalota6; */
												
												
				INSERT INTO cha_ganadores(IdLoteria, IdSorteo, IdConfiguracionSorteo, IdPremioSorteo, idPremio, idJugada, 
																	IdApuesta, NombrePremio, MontoPagar, Estado)
													 SELECT @IdLoteria, parIdSorteo, varIdConfiguracionSorteo, varIdPremioSorteo, varIdPremio, apu.IdJugada, 
																	apu.IdApuesta, varNombrePremio, (apu.MontoApuesta * varMultiplicador) MontoPagar, 'P'
														 FROM cha_apuestas apu
														WHERE apu.IdSorteo = parIdSorteo
														  AND apu.IdPremioSorteo  = varIdPremioSorteo
															AND IF(varIsBalota1, apu.Balota1, '') = IF(varIsBalota1, @sorBalota1, '')
															AND IF(varIsBalota2, apu.Balota2, '') = IF(varIsBalota2, @sorBalota2, '')
															AND IF(varIsBalota3, apu.Balota3, '') = IF(varIsBalota3, @sorBalota3, '')
															AND IF(varIsBalota4, apu.Balota4, '') = IF(varIsBalota4, @sorBalota4, '')
															AND IF(varIsBalota5, apu.Balota5, '') = IF(varIsBalota5, @sorBalota5, '')
															AND IF(varIsBalota6, apu.Balota6, '') = IF(varIsBalota6, @sorBalota6, '')
															AND IF(varIsExtra, apu.Extra, '')     = IF(varIsExtra, @sorExtra, '');
															-- AND apu.IdApuesta NOT IN (SELECT ga.IdApuesta FROM cha_ganadores ga WHERE ga.IdSorteo = parIdSorteo);
												
			ELSE -- COMBINADO
			
			/* SELECT 'COMBINADO', varNombrePremio, varIsBalota1, varIsBalota2, varIsBalota3, varIsBalota4, varIsBalota5, varIsBalota6, 
												varOrdenEstricto, varIsExtra, 
												@sorBalota1, @sorBalota2, @sorBalota3,@sorBalota4,@sorBalota5,@sorBalota6;*/
												
				INSERT INTO cha_ganadores(IdLoteria, IdSorteo, IdConfiguracionSorteo, IdPremioSorteo, idPremio, idJugada, 
																	IdApuesta, NombrePremio, MontoPagar, Estado)
													 SELECT @IdLoteria, parIdSorteo, varIdConfiguracionSorteo, varIdPremioSorteo, varIdPremio, apu.IdJugada, 
																	apu.IdApuesta, varNombrePremio,	(apu.MontoApuesta * varMultiplicador) MontoPagar, 'P'
														 FROM cha_apuestas apu
														WHERE apu.IdSorteo = parIdSorteo
														  AND apu.IdPremioSorteo  = varIdPremioSorteo
															AND (    (IF(varIsBalota1, apu.Balota1, '') = IF(varIsBalota1, @sorBalota1, ''))
																		OR (IF(varIsBalota1, apu.Balota2, '') = IF(varIsBalota1, @sorBalota1, ''))
																		OR (IF(varIsBalota1, apu.Balota3, '') = IF(varIsBalota1, @sorBalota1, ''))
																		OR (IF(varIsBalota1, apu.Balota4, '') = IF(varIsBalota1, @sorBalota1, ''))
																		OR (IF(varIsBalota1, apu.Balota5, '') = IF(varIsBalota1, @sorBalota1, ''))
																		OR (IF(varIsBalota1, apu.Balota6, '') = IF(varIsBalota1, @sorBalota1, ''))
																	) 
															AND (    (IF(varIsBalota2, apu.Balota1, '') = IF(varIsBalota2, @sorBalota2, ''))
																		OR (IF(varIsBalota2, apu.Balota2, '') = IF(varIsBalota2, @sorBalota2, ''))
																		OR (IF(varIsBalota2, apu.Balota3, '') = IF(varIsBalota2, @sorBalota2, ''))
																		OR (IF(varIsBalota2, apu.Balota4, '') = IF(varIsBalota2, @sorBalota2, ''))
																		OR (IF(varIsBalota2, apu.Balota5, '') = IF(varIsBalota2, @sorBalota2, ''))
																		OR (IF(varIsBalota2, apu.Balota6, '') = IF(varIsBalota2, @sorBalota2, ''))
																	) 
															AND (    (IF(varIsBalota3, apu.Balota1, '') = IF(varIsBalota3, @sorBalota3, ''))
																		OR (IF(varIsBalota3, apu.Balota2, '') = IF(varIsBalota3, @sorBalota3, ''))
																		OR (IF(varIsBalota3, apu.Balota3, '') = IF(varIsBalota3, @sorBalota3, ''))
																		OR (IF(varIsBalota3, apu.Balota4, '') = IF(varIsBalota3, @sorBalota3, ''))
																		OR (IF(varIsBalota3, apu.Balota5, '') = IF(varIsBalota3, @sorBalota3, ''))
																		OR (IF(varIsBalota3, apu.Balota6, '') = IF(varIsBalota3, @sorBalota3, ''))
																	) 
															AND (    (IF(varIsBalota4, apu.Balota1, '') = IF(varIsBalota4, @sorBalota4, ''))
																		OR (IF(varIsBalota4, apu.Balota2, '') = IF(varIsBalota4, @sorBalota4, ''))
																		OR (IF(varIsBalota4, apu.Balota3, '') = IF(varIsBalota4, @sorBalota4, ''))
																		OR (IF(varIsBalota4, apu.Balota4, '') = IF(varIsBalota4, @sorBalota4, ''))
																		OR (IF(varIsBalota4, apu.Balota5, '') = IF(varIsBalota4, @sorBalota4, ''))
																		OR (IF(varIsBalota4, apu.Balota6, '') = IF(varIsBalota4, @sorBalota4, ''))
																	) 
															AND (    (IF(varIsBalota5, apu.Balota1, '') = IF(varIsBalota5, @sorBalota5, ''))
																		OR (IF(varIsBalota5, apu.Balota2, '') = IF(varIsBalota5, @sorBalota5, ''))
																		OR (IF(varIsBalota5, apu.Balota3, '') = IF(varIsBalota5, @sorBalota5, ''))
																		OR (IF(varIsBalota5, apu.Balota4, '') = IF(varIsBalota5, @sorBalota5, ''))
																		OR (IF(varIsBalota5, apu.Balota5, '') = IF(varIsBalota5, @sorBalota5, ''))
																		OR (IF(varIsBalota5, apu.Balota6, '') = IF(varIsBalota5, @sorBalota5, ''))
																	) 
															AND (    (IF(varIsBalota6, apu.Balota1, '') = IF(varIsBalota6, @sorBalota6, ''))
																		OR (IF(varIsBalota6, apu.Balota2, '') = IF(varIsBalota6, @sorBalota6, ''))
																		OR (IF(varIsBalota6, apu.Balota3, '') = IF(varIsBalota6, @sorBalota6, ''))
																		OR (IF(varIsBalota6, apu.Balota4, '') = IF(varIsBalota6, @sorBalota6, ''))
																		OR (IF(varIsBalota6, apu.Balota5, '') = IF(varIsBalota6, @sorBalota6, ''))
																		OR (IF(varIsBalota6, apu.Balota6, '') = IF(varIsBalota6, @sorBalota6, ''))
																	) 
															AND IF(varIsExtra, apu.Extra, '') = IF(varIsExtra, @sorExtra, '');
															-- AND apu.IdApuesta NOT IN (SELECT ga.IdApuesta FROM cha_ganadores ga WHERE ga.IdSorteo = parIdSorteo);

			END IF;
			
			
	END LOOP loopQuery;
  CLOSE curPremios;		
	

	-- -------------------------------------------------- 
  COMMIT;
	
END

CREATE DEFINER=`root`@`localhost` PROCEDURE `psProcedimientoEspecialF`(IN pIdJuego INT, IN pTolerancia INT)
sp: BEGIN

-- FECHA      -- AUTOR  -- CAMBIOS
-- 05/01/2023 -- YUSTIZ -- CORREGIDO CARTON PARALELO / CORREGIDO MULTIPLES FIGURAS / AJUSTE CARTON ALEATORIO PARA FIGURAS
-- 11/01/2023 -- YUSTIZ -- CORREGIDO MULTIPLE CONFIGURACIONES
-- 20/01/2023 -- YUSTIZ -- EVITAR CREAR LLENOS ADICIONALES SI FALTAN FIGURAS Y SI NO HA SALIDO LLENO 
-- 14/02/2023 -- YUSTIZ -- AJUSTE DE LLENO CUANDO NO HAY LLENOS ADICIONALES
-- 01/03/2023 -- YUSTIZ -- AGREGADO GANADORES MULTIPLES
-- 04/04/2023 -- YUSTIZ -- AGREGADO LLENOS ADICIONALES MULTIPLES
-- 25/04/2023 -- YUSTIZ -- VERIFICA QUE CARTONES SELECCIONADOS O ALEATORIOS NO HAYAN SALIDO PREVIAMENTE
-- 26/04/2023 -- YUSTIZ -- SELECCIONA UN CARTON DEVUELTO SI EL CARTON ES ALEATORIO ( BINGO V3.8.5+ )
-- 29/06/2023 -- YUSTIZ -- AGREGADO CARTON DUAL AL LLENO
-- 28/07/2023 -- YUSTIZ -- SI ES SOLO VENDIDOS NO SE ACUMULA
-- 27/09/2023 -- YUSTIZ -- SI ES SOLO VENDIDOS Y EL CARTON ES 0 NO SE ACUMULA
-- 29/09/2023 -- YUSTIZ -- FIX 6 CASOS DEL LLENO 
-- 18/10/2023 -- YUSTIZ -- BUSCAR LOS MULTIPLES PRIMERO
-- 25/10/2023 -- YUSTIZ -- VARIOS GANADORES CON UNA BALOTA
-- 13/02/2024 -- YUSTIZ -- AGREGADO CARTON DUAL A FIGURAS
-- 08/03/2024 -- YUSTIZ -- AJUSTE DE TRANSACCION Y OPTIMIZAR QUERYS / EVITAR FALLO CRITICO DEL SP / FIX DUAL MULTIPLE ADELANTADO (VER2)
-- 20/03/2024 -- YUSTIZ -- VALIDAR SI EL CARTON CONFIGURADO NO EXISTE EN RONDASERIE BUSCAR UN CARTON ALEATORIO
-- 21/03/2024 -- YUSTIZ -- AJUSTE DE CARTON DUAL EN LLENO / AJUSTE DUAL ADELANTADO EN VARIAS FIGURAS A LA VEZ
-- 28/11/2024 -- YUSTIZ -- DUALES ADICIONALES DE 2 A 5 FIGURAS Y LLENO
-- 02/12/2024 -- YUSTIZ -- FIX 5 DUALES

DECLARE varDone INT DEFAULT 0;
DECLARE varBalota INT DEFAULT 0;
DECLARE varExisteFigura INT DEFAULT 0;
DECLARE salida VARCHAR(2000);

DECLARE varIdFiguraRonda INT DEFAULT NULL;
DECLARE varIdRonda INT DEFAULT 0;
DECLARE varSoloVendidos VARCHAR(1) DEFAULT 'N';
DECLARE varIdPlenoAutomatico INT DEFAULT NULL;
DECLARE varCarton INT DEFAULT 0;
DECLARE varCartonDual INT DEFAULT NULL;
DECLARE varCartonDual1 INT DEFAULT NULL;
DECLARE varCartonDual2 INT DEFAULT NULL;
DECLARE varCartonDual3 INT DEFAULT NULL;
DECLARE varCartonDual4 INT DEFAULT NULL;
DECLARE varCartonDual5 INT DEFAULT NULL;
DECLARE varMultiple VARCHAR(1) DEFAULT 'N';
DECLARE varContador VARCHAR(1) DEFAULT 'N';

DECLARE curFigurasConf CURSOR FOR (SELECT rf.IdFiguraRonda, pa.IdPlenoAutomatico, rf.Carton, rf.Multiple,
                                          IFNULL(rf.CartonDual, 0),
																					IFNULL(rf.CartonDual2, 0),
																					IFNULL(rf.CartonDual3, 0),
																					IFNULL(rf.CartonDual4, 0),
																					IFNULL(rf.CartonDual5, 0)  
																		 FROM rondasfiguras rf, plenoautomatico pa 
																	  WHERE rf.IdPlenoAutomatico = pa.IdPlenoAutomatico 
																		  AND rf.IdJuego = pIdJuego 
																		  AND rf.IdEstadoRondaFigura = 1 
																		  AND (rf.Acumula = 'S' OR rf.Multiple = 'S') 
																		  AND pa.nombre NOT LIKE '%Lleno %' AND pa.nombre <> 'Lleno' 
															   ORDER BY Multiple DESC);														
												  																
 DECLARE CONTINUE HANDLER FOR NOT FOUND SET varDone = 1;

DECLARE EXIT HANDLER FOR SQLEXCEPTION
  BEGIN
	  GET DIAGNOSTICS CONDITION 1 @sqlst = RETURNED_SQLSTATE, @errno = MYSQL_ERRNO, @texterror = MESSAGE_TEXT;
		SET @texterror = SUBSTR(@texterror,1,100);  -- FIX FALLO CRITICO
    ROLLBACK;
		SELECT @texterror;
END;

	-- datos de la ronda  -- si es solo vendidos no hay acumulados
  SELECT IdRonda, SoloVendidos 
	  INTO varIdRonda, varSoloVendidos
	  FROM rondas 
	 WHERE IdJuego = pIdJuego;
	 
	-- contar la cantidad de balotas jugadas
  SELECT IFNULL(count(1),0) AS Cantidad 
	  INTO @cantBalotas
	  FROM rondashistoriafinal 
	 WHERE IdJuego = pIdJuego 
	   AND Estado = 'A';
 
  -- salgo si aun no existen suficientes balotas segun lo configurado
	IF @cantBalotas < 3 OR @cantBalotas < pTolerancia THEN  
		SET salida := CONCAT('BALOTAS INSUFICIENTES ', @cantBalotas);
		SELECT salida;
	  LEAVE sp;
	END IF;
	
	SELECT IFNULL(Valor, 8)
	  INTO @MinBalotaFigura
	  FROM parametrosgenerales
	 WHERE Referencia = 'MinBalotaFigura';
	
	IF @cantBalotas < @MinBalotaFigura THEN  
		SET salida := CONCAT('BALOTAS INSUFICIENTES SEGUN CONFIGURACION ', @cantBalotas, ' < ' , @MinBalotaFigura);
		SELECT salida;
	  LEAVE sp;
	END IF;
	
  -- buscar primer carton la serie
	SELECT rs.CartonInicial, rs.CartonFinal, rs.Simbolo
	  INTO @CartonInicial, @CartonFinal, @Simbolo
	  FROM rondasseries rs
	 WHERE rs.IdRonda = varIdRonda
	 LIMIT 1;
	
  -- salgo si no existen cartones
	IF @CartonInicial IS NULL THEN
		SET salida := 'VERIFICAR SI EXISTEN CARTONES EN LA TABLA "rondasseries" ';
		SELECT salida;
		LEAVE sp;
	END IF;

	-- salgo si no se juega con los cartones cargados 1 a 1 en rondasseries
	IF @CartonInicial <> @CartonFinal THEN
		SET salida := 'VERIFICAR CARTONES EN LA TABLA "rondasseries" ';
		SELECT salida;
		LEAVE sp;
	END IF;	
	
	SELECT Tabla 
		INTO @TablaSerie 
		FROM listablas 
	 WHERE Simbolo = @Simbolo;

  SET @BuscarSoloMultiples := 'N';
	
	/******************************************************************************************************/
	/******************************************************************************************************/
	/******************************************************************************************************/
	DROP TEMPORARY TABLE IF EXISTS TempDuales;
	 -- Tablas Temporales para iterar en duales   
	CREATE TEMPORARY TABLE TempDuales (IdDual int NOT NULL AUTO_INCREMENT,
                                     IdFiguraRonda int NOT NULL,	
	                                   PosicionDual int NOT NULL,
																		 Carton int DEFAULT NULL, 
																		 PRIMARY KEY (IdDual));
																	 
	/******************************************************************************************************/
	/******************************************************************************************************/
	/******************************************************************************************************/
	
	-- ******************************************
	-- INICIO TRANSACCION
	START TRANSACTION;
	-- ******************************************
	
	-- **** BUSCAR LAS FIGURAS ***** 
	SET @SalidaCallSP := '';
	OPEN curFigurasConf;
	loopQuery: LOOP
  FETCH curFigurasConf INTO varIdFiguraRonda, varIdPlenoAutomatico, varCarton, varMultiple, 
	                          varCartonDual1, varCartonDual2, varCartonDual3, varCartonDual4, varCartonDual5;
		 
		 IF varDone = 1 AND varIdFiguraRonda IS NULL THEN
			  LEAVE loopQuery;
		 END IF;

		 SET @esValido := 1;
		 
		 SET @CartonExisteEnSerie = 0;
		 
		 -- si el carton es diferente a 0 verificar que no haya ganado
		 IF IFNULL(varCarton, 0) > 0 THEN 
		    
			-- verifica si el carton esta rondasseries o si salio anteriormente ***************************************		 
		    SELECT IFNULL(rs.IdSerieRonda, 0), COUNT(hc.IdHistoriaConsulta) 
				  INTO @CartonExisteEnSerie, @CartonFueGanador
	        FROM rondasseries rs
	   LEFT JOIN rondashistoriaconsulta hc ON hc.Carton = rs.CartonInicial AND hc.IdRonda = rs.IdRonda AND EstadoConsulta = 5	  
         WHERE rs.IdRonda = varIdRonda 
	         AND rs.CartonInicial = varCarton;
					 
			 IF @CartonExisteEnSerie = 0 THEN  -- reset carton asignado si no esta importado en rondasseries
					SET varCarton := 0; 
			 END IF;
			 
			 IF @CartonFueGanador > 0 THEN -- reset carton asignado si ya salio anteriormente
					SET varCarton := 0; 
			 END IF;
			 
		 END IF;
		 
		 -- si carton es 0 buscar ALEATORIAMENTE un carton no vendido la serie (que no haya ganado previamente)
		 IF varCarton = 0 THEN 
		 
					 SELECT IFNULL(rs.CartonInicial, -1)
						 INTO varCarton
						 FROM rondasseries rs
						WHERE rs.IdRonda = varIdRonda
							AND rs.Vendido = 'D'-- INSTR(UPPER(rs.Cliente), "NO VENDIDO") > 0
							AND rs.CartonInicial NOT IN ( SELECT Carton 
																		          FROM rondashistoriaconsulta
																	           WHERE IdRonda = rs.IdRonda
																			         AND EstadoConsulta = 5 )
				 ORDER BY RAND() LIMIT 1;
				 
				 IF IFNULL(varSoloVendidos, 'N') = 'S' THEN
					SELECT 'SI SOLO VENDIDOS, NO SE PUEDE ACUMULAR CARTONES EN CERO(0)';
			    SET @CartonAsignado := -1;
			   END IF; 

				 -- salgo si no existen cartones no vendidos
				IF varCarton < 1 THEN
					SELECT 'VERIFICAR SI EXISTEN CARTONES NO VENDIDOS (DEVUELTOS) EN LA TABLA "rondasseries" ';
					ROLLBACK;
					LEAVE sp;
				END IF;
					
		 END IF;
		 
		 SET @CartonAsignado := varCarton;
		 
		 -- no creo ganador si el carton asignado es nulo o menor que cero (0)
		 SET @CartonAsignado := IFNULL(@CartonAsignado, -1);
		 IF @CartonAsignado < 0 THEN
			SET salida := CONCAT('CARTON INVALIDO ( NULL ), FIGURA ', varIdPlenoAutomatico);
			SELECT salida;
			SET @esValido = 0;
		 END IF;
		 
		 SET @EsGanador  := 0;
		 SET @EsMultiple := 0;
		 
		 IF @esValido = 1 THEN
		 
		     IF varMultiple = 'S' THEN 
				     SET @EsMultiple := 1;
				 END IF;
				 
				 IF @BuscarSoloMultiples = 'S' THEN
						SET @EsMultiple := 1;
				 END IF;
				 				 
				 CALL psProcedimientoEspecialF_aux(pIdJuego, 
																						@cantBalotas,
																						varIdFiguraRonda,
																						varIdRonda,
																						varIdPlenoAutomatico,
																						@CartonAsignado,
																						@TablaSerie,
																						0, -- ES LLENO
																						@EsMultiple, -- ES MULTIPLE
																						@EsGanador,
																						@SalidaCallSP);
																						
					SELECT @SalidaCallSP; -- mostrar resultado de cada figura
					
					-- si crea un carton ganador y no es multiple buscara solo multiples
					IF @EsGanador = 1 AND varMultiple = 'N' THEN
					   SET @BuscarSoloMultiples := 'S';
					END IF;
					
					
					/******************************************************************************************************/
					/********************************** DUALES *************************************************************/
					/******************************************************************************************************/
		
		      SET varContador = 1;
					
					WHILE varContador <= 5 DO
					
					  CASE  
						  WHEN (varContador = 1) THEN  
								 SET varCartonDual = varCartonDual1;
						  WHEN (varContador = 2) THEN  
								 SET varCartonDual = varCartonDual2;
						  WHEN (varContador = 3) THEN  
								SET varCartonDual = varCartonDual3;
						  WHEN (varContador = 4) THEN  
								SET varCartonDual = varCartonDual4;
						  WHEN (varContador = 5) THEN  
								SET varCartonDual = varCartonDual5;
						  ELSE 
								SET varCartonDual = 0;
						END CASE;
					
					  
						IF IFNULL(varCartonDual,0) > 0 THEN -- si es dual nunca puede ser cero (0)
					
								 SELECT rs.IdSerieRonda AS ExisteEnSerie, COUNT(hc.IdHistoriaConsulta) AS DualGano, COUNT(tmp.Carton) DualAsignado
									 INTO @CartonDualExisteEnSerie, @CartonDualGano, @DualAsignado
									 FROM rondasseries rs
							LEFT JOIN rondashistoriaconsulta hc ON hc.Carton = rs.CartonInicial AND hc.IdRonda = rs.IdRonda AND EstadoConsulta = 5
							LEFT JOIN TempDuales tmp ON tmp.Carton = rs.CartonInicial
									WHERE rs.IdRonda = varIdRonda 
									  AND rs.CartonInicial = varCartonDual
										AND rs.CartonInicial <> @CartonAsignado;	 -- EL DUAL NO PUEDE SER EL MISMO QUE EL PRINCIPAL
										
										
							 -- SI EL CARTON EXISTE EN RONDASERIE Y NO HA GANADO Y NO SE HA ASIGNADO EN ANTERIORMENTE	
							 IF(IFNULL(@CartonDualExisteEnSerie, 0) > 0 AND IFNULL(@CartonDualGano, 0) < 1 AND IFNULL(@DualAsignado, 0) < 1) THEN
							 
										SET @DualEsMultiple = IF(varMultiple = 'S', 1 , 0);
										
										IF @BuscarSoloMultiples = 'S' THEN -- FIX DUAL ADELANTADO EN VARIAS FIGURAS A LA VEZ
												SET @DualEsMultiple := 1;
										END IF;
												 
										 CALL psProcedimientoEspecialF_aux(pIdJuego, 
																											 @cantBalotas,
																											 varIdFiguraRonda,
																											 varIdRonda,
																											 varIdPlenoAutomatico,
																											 varCartonDual,
																											 @TablaSerie,
																											 0, -- ES LLENO
																											 @DualEsMultiple, -- ES MULTIPLE
																											 @EsGanador,
																											 @SalidaCallSP);
																								
										SELECT @SalidaCallSP; -- mostrar resultado de cada figura
										
										INSERT INTO TempDuales (IdFiguraRonda, PosicionDual, Carton) VALUES (varIdFiguraRonda, varCartonDual, 1);
										
								 ELSE
								 
								  -- indica motivo de la omision del dual (debug)
						      SELECT CONCAT('DUAL ', varCartonDual,' OMITIDO:', 
									              ' EXISTE EN SERIE: ',	  IFNULL(@CartonDualExisteEnSerie, 0),
									              ' GANO ANTERIORMENTE: ', IFNULL(@CartonDualGano, 0) ,
																' ASIGNADO ANTERIORMENTE: ',	IFNULL(@DualAsignado, 0));
								 
							   END IF;
								 
					  END IF;
						SET varCartonDual = 0; -- fix duales null
						SET @CartonDualExisteEnSerie = NULL;
					  SET @CartonDualGano = NULL;
						SET @DualAsignado = NULL;
						SET varContador = varContador + 1;
						
					END WHILE;
					
					
					/******************************************************************************************************/
					/******************************************************************************************************/
					/******************************************************************************************************/
					
					
					
					/************************************************
					-- Si es CARTON  DUAL solicitar segundo carton Ganador
					IF IFNULL(varCartonDual,0) > 0 THEN  -- si es dual nunca puede ser cero (0)
					
						SET @CartonDualFiguraValido := TRUE;

						IF varCartonDual = @CartonAsignado THEN
								SELECT CONCAT('EL CARTON Y DUAL DE LA FIGURA SON EL MISMO ', varIdPlenoAutomatico);
								SET @CartonDualFiguraValido := FALSE;
						END IF;

						 -- verifica si el carton esta rondasseries o si salio anteriormente ***************************************		 
						 SELECT IFNULL(rs.IdSerieRonda, 0), COUNT(hc.IdHistoriaConsulta) 
							 INTO @CartonDualExisteEnSerie, @CartonDualGano
							 FROM rondasseries rs
				  LEFT JOIN rondashistoriaconsulta hc ON hc.Carton = rs.CartonInicial AND hc.IdRonda = rs.IdRonda AND EstadoConsulta = 5	  
						  WHERE rs.IdRonda = varIdRonda 
							  AND rs.CartonInicial = varCartonDual;	 
							 
						 IF @CartonDualExisteEnSerie = 0 THEN  -- reset carton asignado si no esta importado en rondasseries
								SELECT CONCAT('CARTON DUAL NO EXISTE EN SERIE ', varIdPlenoAutomatico);
								SET  @CartonDualFiguraValido := FALSE;
						 END IF;	 
							 	
						 IF IFNULL(@CartonDualGano, 0) > 0 THEN
								SELECT CONCAT('CARTON DUAL YA HA GANADO ANTERIORMENTE ', varIdPlenoAutomatico);
								SET  @CartonDualFiguraValido := FALSE;
						 END IF;
						 						 
						 IF @CartonDualFiguraValido THEN
								 
								 SET @DualEsMultiple = IF(varMultiple = 'S', 1 , 0);
								 
								 IF @BuscarSoloMultiples = 'S' THEN -- FIX DUAL ADELANTADO EN VARIAS FIGURAS A LA VEZ
										SET @DualEsMultiple := 1;
								 END IF;
										 
								 CALL psProcedimientoEspecialF_aux(pIdJuego, 
																									 @cantBalotas,
																									 varIdFiguraRonda,
																									 varIdRonda,
																									 varIdPlenoAutomatico,
																									 varCartonDual,
																									 @TablaSerie,
																									 0, -- ES LLENO
																									 @DualEsMultiple, -- ES MULTIPLE
																									 @EsGanador,
																									 @SalidaCallSP);
																						
								SELECT @SalidaCallSP; -- mostrar resultado de cada figura
						 END IF;
							
					END IF;
					************************************************/
					
     END IF;
		 
		 IF varMultiple = 'N' THEN
				SET varExisteFigura := 1;
		 END IF;


  SET varCartonDual := NULL;
	SET varIdFiguraRonda := NULL; -- fix to not found 
	
	END LOOP loopQuery;
  CLOSE curFigurasConf;	
	
	IF varExisteFigura = 1 THEN
	  COMMIT;
	  LEAVE sp;
	END IF;
	
 -- verifico si aun quedan figuras pendientes, y verifico si falta salir el lleno principal
 SELECT IFNULL(SUM(IF(pa.nombre NOT LIKE '%Lleno %' AND pa.nombre <> 'Lleno', 1, 0)), 0) AS FaltaFiguras,
        IFNULL(SUM(IF(pa.nombre = 'Lleno', 1, 0)), 0) AS FaltaLleno
	 INTO @FaltaFiguras, @FaltaLlenoPpal
   FROM rondasfiguras rf, plenoautomatico pa 
  WHERE rf.IdPlenoAutomatico = pa.IdPlenoAutomatico 
    AND rf.IdJuego = pIdJuego 
    AND rf.IdEstadoRondaFigura = 1; -- figura pendiente
	
	IF @FaltaFiguras > 0 THEN
	  SELECT 'FIGURAS NO ACUMULABLES AUN PENDIENTES';
		COMMIT;
	  LEAVE sp;
	END IF;
	
 -- si no hay acumulable busco el lleno principal configurado
 IF varExisteFigura = 0 THEN
 
		SELECT rf.IdFiguraRonda, pa.IdPlenoAutomatico, co.Carton, co.CartonDual, co.CartonDual2, co.CartonDual3, co.CartonDual4, co.CartonDual5,
		       co.Balotas, IFNULL(pa.IdPlenoAutomatico,0)
			INTO varIdFiguraRonda, varIdPlenoAutomatico, @CartonAsignado, @CartonDual1, @CartonDual2, @CartonDual3, @CartonDual4, @CartonDual5,
			     @BalotasLleno, @EsLLeno 
			FROM rondasfiguras rf, plenoautomatico pa, configuracion co
		 WHERE rf.IdPlenoAutomatico = pa.IdPlenoAutomatico 
			 AND rf.IdJuego = co.NumeroJuego
			 AND rf.IdJuego = pIdJuego
			 AND rf.IdEstadoRondaFigura = 1 
			 AND pa.nombre = 'Lleno'
			 AND co.Estado = 'A'
			 LIMIT 1;
			 
 END IF;
 
 

 IF @FaltaLlenoPpal > 0 AND (varIdFiguraRonda IS NULL OR varIdFiguraRonda = 0) THEN
	  SELECT 'LLENO PRINCIPAL AUN PENDIENTE';
		COMMIT;
	  LEAVE sp;
 END IF;
	 
 -- busco los llenos adicionales 
 IF varIdFiguraRonda IS NULL OR varIdFiguraRonda = 0 THEN
 
    -- se ordena por predecesor y genero cartones de 1 en 1
		SELECT rf.IdFiguraRonda, pa.IdPlenoAutomatico, rf.Carton, rf.CartonDual,rf.CartonDual2, rf.CartonDual3, rf.CartonDual4, rf.CartonDual5,  IFNULL(rf.Multiple, 'N') 
			INTO varIdFiguraRonda, varIdPlenoAutomatico, @CartonAsignado, @CartonDual1, @CartonDual2, @CartonDual3, @CartonDual4, @CartonDual5, @esMultiple 
			FROM rondasfiguras rf, plenoautomatico pa 
		 WHERE rf.IdPlenoAutomatico = pa.IdPlenoAutomatico 
			 AND rf.IdJuego = pIdJuego 
			 AND rf.IdEstadoRondaFigura = 1 
			 AND (rf.Acumula = 'S' OR rf.Multiple = 'S')  
			 AND pa.nombre LIKE '%Lleno %'
	ORDER BY IF(Predecesor = 147, 141.5, Predecesor)  -- FIX orden llenos adicionales  
		 LIMIT 1;
		 
		 -- asigno 76 para que sea multiple, no ganador
		IF @esMultiple = 'S' THEN
			 SET @BalotasLleno := 76;
		END IF;
			 
	END IF;
	
	-- no existen figuras pendientes -> salir
	IF varIdFiguraRonda IS NULL OR varIdFiguraRonda = 0 THEN
			SET salida := 'NO EXISTEN FIGURAS ACUMULABLES';
			SELECT salida;
			COMMIT;
			LEAVE sp;
	END IF;
	 
/***********************************************************/
	-- CREAR LOS LLENOS 
	
	SET @esValido := 1;
	
	 -- si el carton asignado es diferente a 0 verificar que no haya ganado
	 IF IFNULL(@CartonAsignado, 0) > 0 THEN 
				 
		  -- verifica si el carton esta rondasseries o si salio anteriormente ***************************************		 
		  SELECT IFNULL(rs.IdSerieRonda, 0), COUNT(hc.IdHistoriaConsulta) 
			  INTO @CartonExisteEnSerie, @CartonFueGanador
	      FROM rondasseries rs
	 LEFT JOIN rondashistoriaconsulta hc ON hc.Carton = rs.CartonInicial AND hc.IdRonda = rs.IdRonda AND EstadoConsulta = 5	  
       WHERE rs.IdRonda = varIdRonda 
	       AND rs.CartonInicial = @CartonAsignado;
					 
			IF @CartonExisteEnSerie = 0 THEN -- reset carton asignado si no esta importado en rondasseries
					SET @CartonAsignado := 0; 
			END IF;
				 
		  IF @CartonFueGanador > 0 THEN -- reset carton asignado si ya salio anteriormente
				 SET @CartonAsignado := 0; 
		  END IF;
		 
	 END IF;
	 
	 -- si carton es 0 buscar ALEATORIAMENTE un carton no vendido la serie (que no haya ganado previamente)
	 IF @CartonAsignado = 0 THEN 
	 
			 -- SOLO VENDIDOS NO SE PUEDE ACUMULAR CARTONES EN CERO(0)	 
			 IF IFNULL(varSoloVendidos, 'N') = 'S' THEN

					SELECT 'SI SOLO VENDIDOS, NO SE PUEDE ACUMULAR CARTONES EN CERO(0)';
			    SET @CartonAsignado := -1;
					
			 ELSE 
				
					SELECT rs.CartonInicial
					 INTO @CartonInicial
					 FROM rondasseries rs
					WHERE rs.IdRonda = varIdRonda
						AND rs.Vendido = 'D'    -- INSTR(UPPER(rs.Cliente), "NO VENDIDO") > 0
						AND rs.CartonInicial NOT IN ( SELECT Carton 
																					  FROM rondashistoriaconsulta
																					 WHERE IdRonda = rs.IdRonda
																						 AND EstadoConsulta = 5 )
						ORDER BY RAND() LIMIT 1;
				 
				  SET @CartonAsignado := @CartonInicial;

			END IF;	
			 
	 END IF;
	 
	 -- no creo ganador si el carton asignado es nulo o menor que cero (0)
	 SET @CartonAsignado := IFNULL(@CartonAsignado, -1);
	 
	 IF @CartonAsignado < 0 THEN
			SET salida := CONCAT('CARTON INVALIDO ( NULL ), LLENO ', varIdPlenoAutomatico);
			SELECT salida;
			COMMIT;
			LEAVE sp;
	 END IF;
 
	 SET @EsGanador := 0;
	 
	 SELECT CONCAT(varIdFiguraRonda, '  PLENO',varIdPlenoAutomatico,'  CARTON',@CartonAsignado, '  BALOTA',@cantBalotas);
	 
	 
	 CALL psProcedimientoEspecialF_aux(pIdJuego, 
																			@cantBalotas,
																			varIdFiguraRonda,
																			varIdRonda,
																			varIdPlenoAutomatico,
																			@CartonAsignado,
																			@TablaSerie,
																			1, -- ES LLENO
																			0, -- ES MULTIPLE
																			@EsGanador,
																			@SalidaCallSP);
																			
		SELECT @SalidaCallSP; -- mostrar resultado de cada figura
		
	
	/******************************************************************************************************/
	/******************************************************************************************************/
	/******************************************************************************************************/
	DROP TEMPORARY TABLE IF EXISTS TempDuales;
	 -- Tablas Temporales para iterar en duales llenos reinicio ya que el lleno se maneja aprate 
	CREATE TEMPORARY TABLE TempDuales (IdDual int NOT NULL AUTO_INCREMENT,
                                     IdFiguraRonda int NOT NULL,	
	                                   PosicionDual int NOT NULL,
																		 Carton int DEFAULT NULL, 
																		 PRIMARY KEY (IdDual));
																	 
	/******************************************************************************************************/
	/******************************************************************************************************/
	/******************************************************************************************************/


    SELECT 'y EL LLENO DUAL', @CartonDual1, @CartonDual2, @CartonDual3, @CartonDual4, @CartonDual5;
				
	/******************************************************************************************************/
	/********************************** DUALES LLENO *************************************************************/
	/******************************************************************************************************/

	SET varContador = 1;
	
	SET varCartonDual = 0;
	
	WHILE varContador <= 5 DO
	
		CASE  
			WHEN (varContador = 1) THEN  
				 SET varCartonDual = @CartonDual1;
			WHEN (varContador = 2) THEN  
				 SET varCartonDual = @CartonDual2;
			WHEN (varContador = 3) THEN  
				SET varCartonDual =  @CartonDual3;
			WHEN (varContador = 4) THEN  
				SET varCartonDual =  @CartonDual4;
			WHEN (varContador = 5) THEN  
				SET varCartonDual =  @CartonDual5;
			ELSE 
				SET varCartonDual = 0;
		END CASE;
		
		select varContador, varCartonDual;
	
		
		IF IFNULL(@CartonDual,0) > 0 THEN -- si es dual nunca puede ser cero (0)
	
				 SELECT rs.IdSerieRonda AS ExisteEnSerie, COUNT(hc.IdHistoriaConsulta) AS DualGano, COUNT(tmp.Carton) DualAsignado
					 INTO @CartonDualExisteEnSerie, @CartonDualGano, @DualAsignado
					 FROM rondasseries rs
			LEFT JOIN rondashistoriaconsulta hc ON hc.Carton = rs.CartonInicial AND hc.IdRonda = rs.IdRonda AND EstadoConsulta = 5
			LEFT JOIN TempDuales tmp ON tmp.Carton = rs.CartonInicial
					WHERE rs.IdRonda = varIdRonda 
					  AND rs.CartonInicial = varCartonDual
						AND rs.CartonInicial <> @CartonAsignado;	 -- EL DUAL NO PUEDE SER EL MISMO QUE EL PRINCIPAL
						
						
			 -- SI EL CARTON EXISTE EN RONDASERIE Y NO HA GANADO Y NO SE HA ASIGNADO EN ANTERIORMENTE	
			 IF(IFNULL(@CartonDualExisteEnSerie, 0) > 0 AND IFNULL(@CartonDualGano, 0) < 1 AND IFNULL(@DualAsignado, 0) < 1) THEN
			 

						 CALL psProcedimientoEspecialF_aux(pIdJuego, 
																							 @cantBalotas,
																							 varIdFiguraRonda,
																							 varIdRonda,
																							 varIdPlenoAutomatico,
																							 varCartonDual,
																							 @TablaSerie,
																							 1, -- ES LLENO
																							 1, -- ES MULTIPLE
																							 @EsGanador,
																							 @SalidaCallSP);
																				
						SELECT @SalidaCallSP; -- mostrar resultado de cada figura
						
						INSERT INTO TempDuales (IdFiguraRonda, PosicionDual, Carton) VALUES (varIdFiguraRonda, varCartonDual, 1);
						
				 ELSE
				 
					-- indica motivo de la omision del dual (debug)
				  SELECT CONCAT('DUAL LLENO ', varCartonDual,' OMITIDO:', 
												' EXISTE EN SERIE: ',	  IFNULL(@CartonDualExisteEnSerie, 0),
												' GANO ANTERIORMENTE: ', IFNULL(@CartonDualGano, 0) ,
												' ASIGNADO ANTERIORMENTE: ',	IFNULL(@DualAsignado, 0));
				 
				 END IF;
				 
		END IF;
	
		SET varCartonDual = 0; -- fix duales null
		SET @CartonDualExisteEnSerie = NULL;
		SET @CartonDualGano = NULL;
		SET @DualAsignado = NULL;
		SET varContador = varContador + 1;
		
	END WHILE;
	
	
	/******************************************************************************************************/
	/******************************************************************************************************/
	/******************************************************************************************************/
		/*
    -- Si es Lleno  y tiene CARTON  DUAL solicitar segundo carton Ganador
    IF @CartonDual > 0 THEN
		  
			IF @CartonDual = @CartonAsignado THEN
					SET salida := CONCAT('EL CARTON LLENO Y DUAL SON EL MISMO ', varIdPlenoAutomatico);
					SELECT salida;
					COMMIT;
					LEAVE sp;
		  END IF;
			
      -- verifica si el carton esta rondasseries o si salio anteriormente ***************************************		 
			SELECT IFNULL(rs.IdSerieRonda, 0), COUNT(hc.IdHistoriaConsulta) 
				INTO @CartonDualExisteEnSerie, @CartonDualGano
				FROM rondasseries rs
	 LEFT JOIN rondashistoriaconsulta hc ON hc.Carton = rs.CartonInicial AND hc.IdRonda = rs.IdRonda AND EstadoConsulta = 5	  
			 WHERE rs.IdRonda = varIdRonda 
				 AND rs.CartonInicial = @CartonDual;	 
				 
			 IF @CartonDualExisteEnSerie = 0 THEN  -- reset carton asignado si no esta importado en rondasseries
					SELECT CONCAT('CARTON DUAL NO EXISTE EN SERIE ', @CartonDual);
					COMMIT;
					LEAVE sp;
			 END IF;	 
				 
			 IF IFNULL(@CartonDualGano, 0) > 0 THEN
					SELECT CONCAT('CARTON DUAL YA HA GANADO ANTERIORMENTE ', @CartonDual);
					COMMIT;
					LEAVE sp;
		   END IF;
		
		   CALL psProcedimientoEspecialF_aux(pIdJuego, 
																			@cantBalotas,
																			varIdFiguraRonda,
																			varIdRonda,
																			varIdPlenoAutomatico,
																			@CartonDual,
																			@TablaSerie,
																			1, -- ES LLENO
																			1, -- ES MULTIPLE
																			@EsGanador,
																			@SalidaCallSP);
																			
		    SELECT @SalidaCallSP; -- mostrar resultado de cada figura
				
		END IF;
		*/
		
		COMMIT;

END

CREATE DEFINER=`root`@`localhost` PROCEDURE `psProcedimientoEspecialF_aux`(
IN pIdJuego INT, 
IN pCantBalotas INT,
IN pIdFiguraRonda INT,
IN pIdRonda INT,
IN pIdPlenoAutomatico INT,
IN pCartonAsignado INT,
IN pTablaSerie VARCHAR(30),
IN pEsLLeno INT,
IN pEsMultiple INT,
OUT pCreado INT,
OUT pSalida VARCHAR(2000)
)
sp: BEGIN

-- FECHA      -- AUTOR  -- CAMBIOS
-- 06/01/2023 -- YUSTIZ -- CORREGIDO CARTON PARALELO / CORREGIDO MULTIPLES FIGURAS
-- 01/03/2023 -- YUSTIZ -- AGREGADO GANADORES MULTIPLES
-- 10/03/2023 -- YUSTIZ -- EVITA BALOTA DUPLICADA
-- 04/04/2023 -- YUSTIZ -- AGREGADO LLENOS ADICIONALES MULTIPLES 
-- 25/04/2023 -- YUSTIZ -- AJUSTE PARA EVITAR CARTONES/FIGURAS PERFECTAS
-- 20/06/2023 -- YUSTIZ -- GENERA MULTIPLE INCLUSO SI HAY GANADORES NO VENDIDOS
-- 05/07/2023 -- YUSTIZ -- EVITAR CARTONES CON BALOTA 99 (PENDIENTE VERIFICAR CAUSA)
-- 14/07/2023 -- YUSTIZ -- AUDITAR EL ERROR EN BALOTA
-- 27/09/2023 -- YUSTIZ -- EN LLENO NO TOMAR EN CUENTA ULTIMA BALOTA PARA POSIBLES GANADORES
-- 29/09/2023 -- YUSTIZ -- FIX 6 CASOS DEL LLENO
-- 02/10/2023 -- YUSTIZ -- FIX BALOTA 99
-- 25/10/2023 -- YUSTIZ -- REDUCE LA POSIBILIDAD DE QUE SE ADELANTE UN CARTON AL ACUMULADO (VER3)
-- 09/04/2024 -- YUSTIZ -- NUMERO DE BALOTAS ADICIONALES RANDOM (0-2) EN FIGURAS PERFECTAS
-- 07/06/2024 -- YUSTIZ -- REMOVER USO DE QUERY RECURSIVO (WHIT RECURSIVE) POR PROBLEMAS DE PERFORMANCE

DECLARE varDone INT DEFAULT 0;
DECLARE varBalota INT DEFAULT 0;
DECLARE salida VARCHAR(2000);

-- define nro balotas que ya hayan salido, adicionales a la figura, para evitar un carton con figuras perfectas
DECLARE fixFigurasPerfectas INT DEFAULT 2;

DECLARE curBalotas CURSOR FOR  (SELECT balota 
								  FROM rondashistoriafinal 
								 WHERE IdJuego = pIdJuego 
								   AND Estado = 'A'
							  ORDER BY idHistoriaFinal); 

DECLARE CONTINUE HANDLER FOR NOT FOUND SET varDone = 1;

BEGIN
	GET DIAGNOSTICS CONDITION 1 @sqlstate = RETURNED_SQLSTATE, @errno = MYSQL_ERRNO, @text = MESSAGE_TEXT;
END;

  SET pCreado := 0;
	SET @IdRonda := pIdRonda;
	
	-- buscar la figura 
	SELECT F01,F02,F03,F04,F05,F06,F07,F08,F09,F10,
	       F11,F12,F13,F14,F15,F16,F17,F18,F19,
		   F20,F21,F22,F23,F24,F25
	  INTO @F01,@F02,@F03,@F04,@F05,@F06,@F07,@F08,@F09,@F10,
	       @F11,@F12,@F13,@F14,@F15,@F16,@F17,@F18,@F19,
		   @F20,@F21,@F22,@F23,@F24,@F25
   FROM  plenoautomatico pa
  WHERE  pa.IdPlenoAutomatico = pIdPlenoAutomatico;	

	-- contar posiciones ganadoras de la figura, armar filtros de busqueda, indicar columnas 
	SET @CantPosicionesFigura	 := 0;
	SET @PosicionesColumnas := '';
	SET @WherePosiciones := '';
	SET @WhereUltimaBalota := '';
	
	-- verificar si se cumplen las balotas minimas para las posiciones por columnas
	SET @CantPosicionesFigura_b := 0;
	SET @CantPosicionesFigura_i := 0;
	SET @CantPosicionesFigura_n := 0;
	SET @CantPosicionesFigura_g := 0;
	SET @CantPosicionesFigura_o := 0;
	
	IF @F01 = -1 THEN
    SET @CantPosicionesFigura	 = @CantPosicionesFigura	 + 1;
		SET @CantPosicionesFigura_b := @CantPosicionesFigura_b + 1;
		SET @PosicionesColumnas = 'P1,';
    SET @WherePosiciones = '(P1 IN (#B#)) +';
		SET @WhereUltimaBalota = 'OR P1 = #U# ';
	END IF;

	IF @F02 = -1 THEN
			SET @CantPosicionesFigura	 = @CantPosicionesFigura	 + 1;
			SET @CantPosicionesFigura_b := @CantPosicionesFigura_b + 1;
			SET @PosicionesColumnas = CONCAT(@PosicionesColumnas, 'P2,');
			SET @WherePosiciones = CONCAT(@WherePosiciones, '(P2 IN (#B#)) +');
			SET @WhereUltimaBalota = CONCAT(@WhereUltimaBalota, 'OR P2 = #U# ');
	END IF;

	IF @F03 = -1 THEN
			SET @CantPosicionesFigura	 = @CantPosicionesFigura	 + 1;
			SET @CantPosicionesFigura_b := @CantPosicionesFigura_b + 1;
			SET @PosicionesColumnas = CONCAT(@PosicionesColumnas, 'P3,');
			SET @WherePosiciones = CONCAT(@WherePosiciones, '(P3 IN (#B#)) +'); 
			SET @WhereUltimaBalota = CONCAT(@WhereUltimaBalota, 'OR P3 = #U# ');
	END IF;

	IF @F04 = -1 THEN
			SET @CantPosicionesFigura	 = @CantPosicionesFigura	 + 1;
			SET @CantPosicionesFigura_b := @CantPosicionesFigura_b + 1;
			SET @PosicionesColumnas = CONCAT(@PosicionesColumnas, 'P4,');
			SET @WherePosiciones = CONCAT(@WherePosiciones, '(P4 IN (#B#)) +'); 
			SET @WhereUltimaBalota = CONCAT(@WhereUltimaBalota, 'OR P4 = #U# ');
	END IF;

	IF @F05 = -1 THEN
			SET @CantPosicionesFigura	 = @CantPosicionesFigura	 + 1;
			SET @CantPosicionesFigura_b := @CantPosicionesFigura_b + 1;
			SET @PosicionesColumnas = CONCAT(@PosicionesColumnas, 'P5,');
			SET @WherePosiciones = CONCAT(@WherePosiciones, '(P5 IN (#B#)) +');
		SET @WhereUltimaBalota = CONCAT(@WhereUltimaBalota, 'OR P5 = #U# ');	
	END IF;

	IF @F06 = -1 THEN
			SET @CantPosicionesFigura	 = @CantPosicionesFigura	 + 1;
			SET @CantPosicionesFigura_i := @CantPosicionesFigura_i + 1;
			SET @PosicionesColumnas = CONCAT(@PosicionesColumnas, 'P6,');
			SET @WherePosiciones = CONCAT(@WherePosiciones, '(P6 IN (#I#)) +'); 
			SET @WhereUltimaBalota = CONCAT(@WhereUltimaBalota, 'OR P6 = #U# ');
	END IF;

	IF @F07 = -1 THEN
			SET @CantPosicionesFigura	 = @CantPosicionesFigura	 + 1;
			SET @CantPosicionesFigura_i := @CantPosicionesFigura_i + 1;
			SET @PosicionesColumnas = CONCAT(@PosicionesColumnas, 'P7,');
			SET @WherePosiciones = CONCAT(@WherePosiciones, '(P7 IN (#I#)) +');
		    SET @WhereUltimaBalota = CONCAT(@WhereUltimaBalota, 'OR P7 = #U# ');	
	END IF;

	IF @F08 = -1 THEN
			SET @CantPosicionesFigura	 = @CantPosicionesFigura	 + 1;
			SET @CantPosicionesFigura_i := @CantPosicionesFigura_i + 1;
			SET @PosicionesColumnas = CONCAT(@PosicionesColumnas, 'P8,');
			SET @WherePosiciones = CONCAT(@WherePosiciones, '(P8 IN (#I#)) +');
		    SET @WhereUltimaBalota = CONCAT(@WhereUltimaBalota, 'OR P8 = #U# ');	
	END IF;

	IF @F09 = -1 THEN
			SET @CantPosicionesFigura	 = @CantPosicionesFigura	 + 1;
			SET @CantPosicionesFigura_i := @CantPosicionesFigura_i + 1;
			SET @PosicionesColumnas = CONCAT(@PosicionesColumnas, 'P9,');
			SET @WherePosiciones = CONCAT(@WherePosiciones, '(P9 IN (#I#)) +');
		    SET @WhereUltimaBalota = CONCAT(@WhereUltimaBalota, 'OR P9 = #U# ');	
	END IF;

	IF @F10 = -1 THEN
			SET @CantPosicionesFigura	 = @CantPosicionesFigura	 + 1;
			SET @CantPosicionesFigura_i := @CantPosicionesFigura_i + 1;
			SET @PosicionesColumnas = CONCAT(@PosicionesColumnas, 'P10,');
			SET @WherePosiciones = CONCAT(@WherePosiciones, '(P10 IN (#I#)) +');
		  SET @WhereUltimaBalota = CONCAT(@WhereUltimaBalota, 'OR P10 = #U# ');	
	END IF;

	IF @F11 = -1 THEN
			SET @CantPosicionesFigura	 = @CantPosicionesFigura	 + 1;
			SET @CantPosicionesFigura_n := @CantPosicionesFigura_n + 1;
			SET @PosicionesColumnas = CONCAT(@PosicionesColumnas, 'P11,');
			SET @WherePosiciones = CONCAT(@WherePosiciones, '(P11 IN (#N#)) +');
		    SET @WhereUltimaBalota = CONCAT(@WhereUltimaBalota, 'OR P11 = #U# ');	
	END IF;

	IF @F12 = -1 THEN
			SET @CantPosicionesFigura	 = @CantPosicionesFigura	 + 1;
			SET @CantPosicionesFigura_n := @CantPosicionesFigura_n + 1;
			SET @PosicionesColumnas = CONCAT(@PosicionesColumnas, 'P12,');
			SET @WherePosiciones = CONCAT(@WherePosiciones, '(P12 IN (#N#)) +'); 
			SET @WhereUltimaBalota = CONCAT(@WhereUltimaBalota, 'OR P12 = #U# ');
	END IF;

	IF @F13 = -1 THEN
			SET @CantPosicionesFigura	 = @CantPosicionesFigura	 + 1;
			SET @CantPosicionesFigura_n := @CantPosicionesFigura_n + 1;
			SET @PosicionesColumnas = CONCAT(@PosicionesColumnas, 'P13,');
			SET @WherePosiciones = CONCAT(@WherePosiciones, '(P13 IN (#N#)) +'); 
			SET @WhereUltimaBalota = CONCAT(@WhereUltimaBalota, 'OR P13 = #U# ');
	END IF;

	IF @F14 = -1 THEN
			SET @CantPosicionesFigura	 = @CantPosicionesFigura	 + 1;
			SET @CantPosicionesFigura_n := @CantPosicionesFigura_n + 1;
			SET @PosicionesColumnas = CONCAT(@PosicionesColumnas, 'P14,');
			SET @WherePosiciones = CONCAT(@WherePosiciones, '(P14 IN (#N#)) +'); 
			SET @WhereUltimaBalota = CONCAT(@WhereUltimaBalota, 'OR P14 = #U# ');
	END IF;

	IF @F15 = -1 THEN
			SET @CantPosicionesFigura	 = @CantPosicionesFigura	 + 1;
			SET @CantPosicionesFigura_n := @CantPosicionesFigura_n + 1;
			SET @PosicionesColumnas = CONCAT(@PosicionesColumnas, 'P15,');
			SET @WherePosiciones = CONCAT(@WherePosiciones, '(P15 IN (#N#)) +'); 
			SET @WhereUltimaBalota = CONCAT(@WhereUltimaBalota, 'OR P15 = #U# ');
	END IF;

	IF @F16 = -1 THEN
			SET @CantPosicionesFigura	 = @CantPosicionesFigura	 + 1;
			SET @CantPosicionesFigura_g := @CantPosicionesFigura_g + 1;
			SET @PosicionesColumnas = CONCAT(@PosicionesColumnas, 'P16,');
			SET @WherePosiciones = CONCAT(@WherePosiciones, '(P16 IN (#G#)) +'); 
			SET @WhereUltimaBalota = CONCAT(@WhereUltimaBalota, 'OR P16 = #U# ');
	END IF;

	IF @F17 = -1 THEN
			SET @CantPosicionesFigura	 = @CantPosicionesFigura	 + 1;
			SET @CantPosicionesFigura_g := @CantPosicionesFigura_g + 1;
			SET @PosicionesColumnas = CONCAT(@PosicionesColumnas, 'P17,');
			SET @WherePosiciones = CONCAT(@WherePosiciones, '(P17 IN (#G#)) +'); 
			SET @WhereUltimaBalota = CONCAT(@WhereUltimaBalota, 'OR P17 = #U# ');
	END IF;

	IF @F18 = -1 THEN
			SET @CantPosicionesFigura	 = @CantPosicionesFigura	 + 1;
			SET @CantPosicionesFigura_g := @CantPosicionesFigura_g + 1;
			SET @PosicionesColumnas = CONCAT(@PosicionesColumnas, 'P18,');
			SET @WherePosiciones = CONCAT(@WherePosiciones, '(P18 IN (#G#)) +'); 
			SET @WhereUltimaBalota = CONCAT(@WhereUltimaBalota, 'OR P18 = #U# ');
	END IF;

	IF @F19 = -1 THEN
			SET @CantPosicionesFigura	 = @CantPosicionesFigura	 + 1;
			SET @CantPosicionesFigura_g := @CantPosicionesFigura_g + 1;
			SET @PosicionesColumnas = CONCAT(@PosicionesColumnas, 'P19,');
			SET @WherePosiciones = CONCAT(@WherePosiciones, '(P19 IN (#G#)) +'); 
			SET @WhereUltimaBalota = CONCAT(@WhereUltimaBalota, 'OR P19 = #U# ');
	END IF;

	IF @F20 = -1 THEN
			SET @CantPosicionesFigura	 = @CantPosicionesFigura	 + 1;
			SET @CantPosicionesFigura_g := @CantPosicionesFigura_g + 1;
			SET @PosicionesColumnas = CONCAT(@PosicionesColumnas, 'P20,');
			SET @WherePosiciones = CONCAT(@WherePosiciones, '(P20 IN (#G#)) +');
		  SET @WhereUltimaBalota = CONCAT(@WhereUltimaBalota, 'OR P20 = #U# ');	
	END IF;

	IF @F21 = -1 THEN
			SET @CantPosicionesFigura	 = @CantPosicionesFigura	 + 1;
			SET @CantPosicionesFigura_o := @CantPosicionesFigura_o + 1;
			SET @PosicionesColumnas = CONCAT(@PosicionesColumnas, 'P21,');
			SET @WherePosiciones = CONCAT(@WherePosiciones, '(P21 IN (#O#)) +'); 
			SET @WhereUltimaBalota = CONCAT(@WhereUltimaBalota, 'OR P21 = #U# ');
	END IF;

	IF @F22 = -1 THEN
			SET @CantPosicionesFigura	 = @CantPosicionesFigura	 + 1;
			SET @CantPosicionesFigura_o := @CantPosicionesFigura_o + 1;
			SET @PosicionesColumnas = CONCAT(@PosicionesColumnas, 'P22,');
			SET @WherePosiciones = CONCAT(@WherePosiciones, '(P22 IN (#O#)) +');
		  SET @WhereUltimaBalota = CONCAT(@WhereUltimaBalota, 'OR P22 = #U# ');	
	END IF;

	IF @F23 = -1 THEN
			SET @CantPosicionesFigura	 = @CantPosicionesFigura	 + 1;
			SET @CantPosicionesFigura_o := @CantPosicionesFigura_o + 1;
			SET @PosicionesColumnas = CONCAT(@PosicionesColumnas, 'P23,');
			SET @WherePosiciones = CONCAT(@WherePosiciones, '(P23 IN (#O#)) +'); 
			SET @WhereUltimaBalota = CONCAT(@WhereUltimaBalota, 'OR P23 = #U# ');
	END IF;

	IF @F24 = -1 THEN
			SET @CantPosicionesFigura	 = @CantPosicionesFigura	 + 1;
			SET @CantPosicionesFigura_o := @CantPosicionesFigura_o + 1;
			SET @PosicionesColumnas = CONCAT(@PosicionesColumnas, 'P24,');
			SET @WherePosiciones = CONCAT(@WherePosiciones, '(P24 IN (#O#)) +');
		  SET @WhereUltimaBalota = CONCAT(@WhereUltimaBalota, 'OR P24 = #U# ');	
	END IF;

	IF @F25 = -1 THEN
			SET @CantPosicionesFigura	 = @CantPosicionesFigura + 1;
			SET @CantPosicionesFigura_o := @CantPosicionesFigura_o + 1;
			SET @PosicionesColumnas = CONCAT(@PosicionesColumnas, 'P25,');
			SET @WherePosiciones = CONCAT(@WherePosiciones, '(P25 IN (#O#)) +'); 
			SET @WhereUltimaBalota = CONCAT(@WhereUltimaBalota, 'OR P25 = #U# ');
	END IF;
	
	-- salir si aun no se cumple minimo de balotas para las posiciones activas
	IF pCantBalotas < (@CantPosicionesFigura - 1) THEN
		SET salida := concat('BALOTAS ', pCantBalotas, ' INSUFICIENTES PARA POSICIONES ', @CantPosicionesFigura - 1, ' FIGURA ', pIdPlenoAutomatico);
		SET pSalida := salida;
		LEAVE sp;
	END IF;
	 
	SET @balotas_b := '99';
	SET @balotas_i := '99';
	SET @balotas_n := '99';
	SET @balotas_g := '99';
	SET @balotas_o := '99';
	
	SET @CantBalotas_b := 0;
	SET @CantBalotas_i := 0;
	SET @CantBalotas_n := 0;
	SET @CantBalotas_g := 0;
	SET @CantBalotas_o := 0;
	
	-- buscar balotas que ya han salido por letra 
	OPEN curBalotas;
	loopQuery: LOOP
   FETCH curBalotas INTO varBalota;
		 IF varDone = 1 THEN
			  LEAVE loopQuery;
		 END IF;
		 
		 CASE  
		   WHEN (varBalota BETWEEN 01 AND 15) THEN 
			   SET @balotas_b := CONCAT(@balotas_b ,',', LPAD(varBalota, 2, '0'));
			   SET @CantBalotas_b := @CantBalotas_b + 1;
		   WHEN (varBalota BETWEEN 16 AND 30) THEN 
			   SET @balotas_i := CONCAT(@balotas_i ,',', varBalota);
			   SET @CantBalotas_i := @CantBalotas_i + 1;
		   WHEN (varBalota BETWEEN 31 AND 45) THEN 
			   SET @balotas_n := CONCAT(@balotas_n ,',', varBalota);
				 SET @CantBalotas_n := @CantBalotas_n + 1;
		   WHEN (varBalota BETWEEN 46 AND 60) THEN 
			   SET @balotas_g := CONCAT(@balotas_g ,',', varBalota);
				 SET @CantBalotas_g := @CantBalotas_g + 1;
		   WHEN (varBalota BETWEEN 61 AND 75) THEN 
			   SET @balotas_o := CONCAT(@balotas_o ,',', varBalota);
				 SET @CantBalotas_o := @CantBalotas_o + 1;
     END CASE;
		 
	END LOOP loopQuery;
  CLOSE curBalotas;	
	
	-- salgo si no se cumple el numero de balotas para cualquier columna
	IF @CantBalotas_b < @CantPosicionesFigura_b OR
	   @CantBalotas_i < @CantPosicionesFigura_i OR
	   @CantBalotas_n < @CantPosicionesFigura_n OR
	   @CantBalotas_g < @CantPosicionesFigura_g OR
	   @CantBalotas_o < @CantPosicionesFigura_o THEN
		   SET salida := CONCAT('FIGURA ', pIdPlenoAutomatico, ' BALOTAS INSUFICIENTES PARA ALGUNAS POSICIONES POR LETRAS ', 
			                      ' B: ',@CantPosicionesFigura_b, ' > ' , @CantBalotas_b, '-',@balotas_b,
														' I: ',@CantPosicionesFigura_i, ' > ' , @CantBalotas_i, '-',@balotas_i,
														' N: ',@CantPosicionesFigura_n, ' > ' , @CantBalotas_n, '-',@balotas_n,
														' G: ',@CantPosicionesFigura_g, ' > ' , @CantBalotas_g, '-',@balotas_g,
														' O: ',@CantPosicionesFigura_o, ' > ' , @CantBalotas_o, '-',@balotas_o
														);
			 SET pSalida := salida;
		   LEAVE sp;
  END IF;
	
	 -- formateo el where de las posiciones ganadoras
	 SET @WherePosiciones := REPLACE(@WherePosiciones, '#B#', @balotas_b);
	 SET @WherePosiciones := REPLACE(@WherePosiciones, '#I#', @balotas_i);
	 SET @WherePosiciones := REPLACE(@WherePosiciones, '#N#', @balotas_n);
	 SET @WherePosiciones := REPLACE(@WherePosiciones, '#G#', @balotas_g);
	 SET @WherePosiciones := REPLACE(@WherePosiciones, '#O#', @balotas_o);
	 SET @WherePosiciones := SUBSTR(@WherePosiciones, 1, length(@WherePosiciones)-1); -- remove ultimo +
	 
	 -- formateo el where para que se tome la ultima balota
	 SET @WhereUltimaBalota := SUBSTR(@WhereUltimaBalota, 3); -- remove primer OR
	 SET @WhereUltimaBalota := REPLACE(@WhereUltimaBalota, '#U#', varBalota);
	 
   -- BUSCO POSIBLES GANADORES 
	 SET @MaximasPosicionesPermitidas := @CantPosicionesFigura - 1 + pEsMultiple;
	 SET @cantPosiblesGanadoresNoVendidos := 0;
	 SET @cantPosiblesGanadoresVendidos := 0;
		
	 -- hacer la busqueda de posibles ganadores
	 SET @sqlstr :=	CONCAT('SELECT SUM(IF(INSTR(rs.Cliente, ''NO VENDIDO'') > 0, 1, 0 )) AS NoVendidos, 
																 SUM(IF(INSTR(rs.Cliente, ''NO VENDIDO'') = 0, 1, 0 )) AS Vendidos
													  INTO @cantPosiblesGanadoresNoVendidos, 
																 @cantPosiblesGanadoresVendidos
														FROM rondasseries rs
														JOIN ', pTablaSerie , ' se ON se.Numero = rs.CartonInicial
													 WHERE rs.IdRonda = @IdRonda 
														 AND (' , @WherePosiciones, ') >= ', @MaximasPosicionesPermitidas, 
													 ' AND ( ', IF(pEsLLeno = 1 OR pEsMultiple = 0 , '1 = 1', @WhereUltimaBalota ), ')' 
												 );
		
		select concat('ganador ', @sqlstr);
		PREPARE stmt FROM @sqlstr;
		EXECUTE stmt;
		DEALLOCATE PREPARE stmt;
		
		SET @cantPosiblesGanadoresNoVendidos := IFNULL(@cantPosiblesGanadoresNoVendidos,0);
		SET @cantPosiblesGanadoresVendidos := IFNULL(@cantPosiblesGanadoresVendidos,0);
		
		--  VERIFICAR EL LLENO Y CARTON PARALEL0 
			-- acumulado simple   balota fija 
			-- acumulado auto     balota 0
			-- acumulado multiple paralelo  balota 76
			-- dual               dos cartones acumulados balota fija
			-- dual auto          dos cartones acumulados balota 0
			-- dual multiple      dos cartones paralelos balota 76 
		
		-- si la figura es lleno y esta configurado verifico si existe un ganador para crear un ganador paralelo 
		IF pEsLLeno = 1 THEN
		  SET @BalotasLleno  := IFNULL(@BalotasLleno,0);
		  SET @esParalelo := (@BalotasLleno = 76);
			SET @cantPosiblesGanadoresVendidosParalelo := 0;
			SET @cantPosiblesGanadoresNoVendidosParalelo := 0;
		
			SET @sqlstr :=	CONCAT('SELECT SUM(IF(INSTR(rs.Cliente, ''NO VENDIDO'') > 0, 1, 0 )) AS NoVendidos,
			                               SUM(IF(INSTR(rs.Cliente, ''NO VENDIDO'') = 0, 1, 0 )) AS Vendidos
													  INTO @cantPosiblesGanadoresNoVendidosParalelo,
														     @cantPosiblesGanadoresVendidosParalelo
														FROM rondasseries rs
														JOIN ', pTablaSerie , ' se ON se.Numero = rs.CartonInicial
													 WHERE rs.IdRonda = @IdRonda 
													   AND (' , @WherePosiciones, ') = ', @CantPosicionesFigura, 
													 ' AND ( ', @WhereUltimaBalota, ')' 
												 );
												 
			-- select concat('paralelo ', @sqlstr);
			PREPARE stmt FROM @sqlstr;
			EXECUTE stmt;
			DEALLOCATE PREPARE stmt;
			
      SET @cantPosiblesGanadoresNoVendidosParalelo := IFNULL(@cantPosiblesGanadoresNoVendidosParalelo,0);
			SET @cantPosiblesGanadoresVendidosParalelo := IFNULL(@cantPosiblesGanadoresVendidosParalelo,0);
			
			SET @PosiblesGanadoresLLeno =  @cantPosiblesGanadoresNoVendidos + @cantPosiblesGanadoresVendidos;
			SET @PosiblesGanadoresParalelo = @cantPosiblesGanadoresNoVendidosParalelo + @cantPosiblesGanadoresVendidosParalelo;
			
		  -- si no existe un posible ganador (lleno y paralelo)
			IF @PosiblesGanadoresLLeno < 1 AND @PosiblesGanadoresParalelo < 1 THEN
				   
					SELECT CONCAT(' NO EXISTE GANADOR L', @PosiblesGanadoresLLeno, ' P', @PosiblesGanadoresParalelo);
					SELECT CONCAT('  @BalotasLleno ',  @BalotasLleno, ' pCantBalotas', pCantBalotas);
				
			    -- si esta configurado balota fija y es mayo o igual a la balota actual	
					IF @BalotasLleno > 0 AND @BalotasLleno <= pCantBalotas THEN
							SET @cantPosiblesGanadoresVendidos := 1;
					END IF;
					
					-- si esta configurado balota fija pero aun no se cumple las balotas necesarias
					IF @BalotasLleno > 0 AND @BalotasLleno > pCantBalotas THEN
							SET salida := concat ('BALOTAS INSUFICIENTES PARA EL LLENO SEGUN CONFIGURACION ', pCantBalotas,' < ', @BalotasLleno);
							SET pSalida := salida;
							LEAVE sp;
					END IF;
					    
			ELSE -- si existe posible ganador obligar minimo ganador paralelo
			
			    SELECT CONCAT('EXISTE GANADOR L', @PosiblesGanadoresLLeno, ' P', @PosiblesGanadoresParalelo);
			    SELECT CONCAT('pCantBalotas ',pCantBalotas,' @PosiblesGanadoresParalelo ', @PosiblesGanadoresParalelo);
			    
					-- si balotas es 0 espero a la balota minima para el lleno segun la configuracion
					IF @BalotasLleno = 0 THEN 
						 SELECT IFNULL(Valor, 45)
							 INTO @MinBalotaLleno
							 FROM parametrosgenerales
							WHERE Referencia = 'MinBalotaLleno';
							
						 -- si no existe un ganador y aun no se llega a la minima segun parametrosGenerales
						 IF pCantBalotas < @MinBalotaLleno  AND @PosiblesGanadoresParalelo < 1 THEN 
								SET salida := CONCAT('BALOTAS INSUFICIENTES PARA EL LLENO SEGUN PARAMETROS ', pCantBalotas, ' < ' , @MinBalotaLleno);
								SET pSalida := salida;
								LEAVE sp;
						 END IF;
					END IF;
			
          SET @cantPosiblesGanadoresVendidos := 1;
					
					-- si es paralelo espero a que exista un ganador efectivo
					IF @esParalelo AND @PosiblesGanadoresParalelo < 1 THEN 
						 SET @cantPosiblesGanadoresVendidos := 0;
						 SET @cantPosiblesGanadoresNoVendidos := 0;
					END IF;
					
			END IF; -- fin de existe ganador
		
		END IF; -- fin de lleno
		
    -- salir si no existe posibilidad de ganadores
		--  busco vendidos y no vendidos (serie completa)
		IF @cantPosiblesGanadoresVendidos < 1 AND  @cantPosiblesGanadoresNoVendidos < 1  THEN
			SET salida := concat('FIGURA ', pIdPlenoAutomatico, 'NO HAY POSIBLES GANADORES ', @cantPosiblesGanadoresVendidos);
			SET pSalida := salida;
			LEAVE sp;
		END IF;
	
	  select concat('vendidos ' , @cantPosiblesGanadoresVendidos, ' no vendidos ' , @cantPosiblesGanadoresNoVendidos, ' carton ' ,  pCartonAsignado);
		select concat('paralelo ' , @PosiblesGanadoresParalelo, ' carton ' , pCartonAsignado, ' balotalleno ', @BalotasLleno);
		
	-- EXISTE POSIBILIDAD DE GANADOR ASI QUE GENERO UN CARTON GANADOR
	-- CASO 1: GENERO UN NUEVO CARTON CON LAS BALOTAS GANADORAS
	
	
	 -- Eliminar objetos temporales.
  DROP TEMPORARY TABLE IF EXISTS TempNumerosCarton;
   
  -- Tablas Temporales de numero de hojas    
  CREATE TEMPORARY TABLE TempNumerosCarton ( Numero int NOT NULL, PRIMARY KEY (Numero));
	
	-- Lleno la tabla con los numero de un carton 1-75 (elimina query recursivo WHITH RECURSIVE)
	INSERT INTO TempNumerosCarton (Numero)
		VALUES (1),(2),(3),(4),(5),(6),(7),(8),(9),(10),(11),(12),(13),(14),(15),
		      (16),(17),(18),(19),(20),(21),(22),(23),(24),(25),(26),(27),(28),(29),(30),
					(31),(32),(33),(34),(35),(36),(37),(38),(39),(40),(41),(42),(43),(44),(45),
					(46),(47),(48),(49),(50),(51),(52),(53),(54),(55),(56),(57),(58),(59),(60),
					(61),(62),(63),(64),(65),(66),(67),(68),(69),(70),(71),(72),(73),(74),(75);    
	
	-- busco la letra de la ultima balota para la asignacion obligatoria
	SET @UltimaBalota := varBalota;
	CASE  
		 WHEN (@UltimaBalota BETWEEN 01 AND 15) THEN SET @UltimaBalotaLetra := 'B';
		 WHEN (@UltimaBalota BETWEEN 16 AND 30) THEN SET @UltimaBalotaLetra := 'I'; 
		 WHEN (@UltimaBalota BETWEEN 31 AND 45) THEN SET @UltimaBalotaLetra := 'N'; 
		 WHEN (@UltimaBalota BETWEEN 46 AND 60) THEN SET @UltimaBalotaLetra := 'G'; 
		 WHEN (@UltimaBalota BETWEEN 61 AND 75) THEN SET @UltimaBalotaLetra := 'O'; 
	END CASE;
	
	-- limpio balotas por letra
	SET @balotas_b := SUBSTR(@balotas_b, 4); -- REMOVE 99,
	SET @balotas_i := SUBSTR(@balotas_i, 4); -- REMOVE 99,
	SET @balotas_n := SUBSTR(@balotas_n, 4); -- REMOVE 99,
	SET @balotas_g := SUBSTR(@balotas_g, 4); -- REMOVE 99,
	SET @balotas_o := SUBSTR(@balotas_o, 4); -- REMOVE 99,
	
	SET @numColums := 1;
	SET @balotasSeleccionadas := '99,';
	SET @balotaSeleccionada := '';
	SET @balotasUpd = '';
	
	-- calcula el numero de balotas ramdom para el fix de la figura perfecta
	SET fixFigurasPerfectas = FLOOR(RAND()*3); -- random  >= 0 and <= 2;
	
	-- selecciono cual columna(letra) y posicion tiene mas balotas DISPONIBLES para agregar balotas adicionales a la figura en cartones perfectos
	    
	-- si tiene mas de 4 balotas la columna las dejo en 0 para que no se tomen en cuenta
	SET @balotasDisponibles_b := CASE WHEN @CantPosicionesFigura_b > 4 THEN 0 ELSE (@CantBalotas_b - @CantPosicionesFigura_b) END;
	SET @balotasDisponibles_i := CASE WHEN @CantPosicionesFigura_i > 4 THEN 0 ELSE (@CantBalotas_i - @CantPosicionesFigura_i) END;
	SET @balotasDisponibles_n := CASE WHEN @CantPosicionesFigura_n > 3 THEN 0 ELSE (@CantBalotas_n - @CantPosicionesFigura_n) END;
	SET @balotasDisponibles_g := CASE WHEN @CantPosicionesFigura_g > 4 THEN 0 ELSE (@CantBalotas_g - @CantPosicionesFigura_g) END;
	SET @balotasDisponibles_o := CASE WHEN @CantPosicionesFigura_o > 4 THEN 0 ELSE (@CantBalotas_o - @CantPosicionesFigura_o) END;
	
	WITH Columnas AS (
						SELECT 'B' AS LETRA, @balotasDisponibles_b AS CANTIDAD, (SELECT P FROM (
							SELECT IF(@F01 = 0, 1, 99) AS P UNION 
							SELECT IF(@F02 = 0, 2, 99) AS P UNION  
							SELECT IF(@F03 = 0, 3, 99) AS P UNION  
							SELECT IF(@F04 = 0, 4, 99) AS P UNION  
							SELECT IF(@F05 = 0, 5, 99)
						) POS WHERE P <> 99 ORDER BY RAND() LIMIT 1 ) AS POSICION
						UNION 
						SELECT 'I' AS LETRA, @balotasDisponibles_i AS CANTIDAD, (SELECT P FROM (
							SELECT IF(@F06 = 0, 6, 99) AS P UNION 
							SELECT IF(@F07 = 0, 7, 99) AS P UNION  
							SELECT IF(@F08 = 0, 8, 99) AS P UNION  
							SELECT IF(@F09 = 0, 9, 99) AS P UNION  
							SELECT IF(@F10 = 0, 10, 99)
						) POS WHERE P <> 99 ORDER BY RAND() LIMIT 1 ) AS POSICION
						UNION
						SELECT 'N' AS LETRA, @balotasDisponibles_n AS CANTIDAD, (SELECT P FROM (
							SELECT IF(@F11 = 0, 11, 99) AS P UNION 
							SELECT IF(@F12 = 0, 12, 99) AS P UNION  
							SELECT IF(@F13 = 0, 13, 99) AS P UNION  
							SELECT IF(@F14 = 0, 14, 99) AS P UNION  
							SELECT IF(@F15 = 0, 15, 99)
						) POS WHERE P <> 99 ORDER BY RAND() LIMIT 1 ) AS POSICION
						UNION
						SELECT 'G' AS LETRA, @balotasDisponibles_g AS CANTIDAD, (SELECT P FROM (
							SELECT IF(@F16 = 0, 16, 99) AS P UNION 
							SELECT IF(@F17 = 0, 17, 99) AS P UNION  
							SELECT IF(@F18 = 0, 18, 99) AS P UNION  
							SELECT IF(@F19 = 0, 19, 99) AS P UNION  
							SELECT IF(@F20 = 0, 20, 99)
						) POS WHERE P <> 99 ORDER BY RAND() LIMIT 1 ) AS POSICION
						UNION
						SELECT 'O' AS LETRA, @balotasDisponibles_o AS CANTIDAD, (SELECT P FROM (
							SELECT IF(@F21 = 0, 21, 99) AS P UNION 
							SELECT IF(@F22 = 0, 22, 99) AS P UNION  
							SELECT IF(@F23 = 0, 23, 99) AS P UNION  
							SELECT IF(@F24 = 0, 24, 99) AS P UNION  
							SELECT IF(@F25 = 0, 25, 99)
						) POS WHERE P <> 99 ORDER BY RAND() LIMIT 1 ) AS POSICION
					)									
					SELECT				
						(SELECT LETRA FROM Columnas WHERE CANTIDAD > 0 ORDER BY CANTIDAD DESC LIMIT 1  ),
						(SELECT LETRA FROM Columnas WHERE CANTIDAD > 0 ORDER BY CANTIDAD DESC LIMIT 1, 1),
						(SELECT CANTIDAD FROM Columnas WHERE CANTIDAD > 0 ORDER BY CANTIDAD DESC LIMIT 1  ),
						(SELECT CANTIDAD FROM Columnas WHERE CANTIDAD > 0 ORDER BY CANTIDAD DESC LIMIT 1, 1),
						(SELECT POSICION FROM Columnas WHERE CANTIDAD > 0 ORDER BY CANTIDAD DESC LIMIT 1  ),
						(SELECT POSICION FROM Columnas WHERE CANTIDAD > 0 ORDER BY CANTIDAD DESC LIMIT 1, 1)
					INTO 
					@primeraLetraFix, @segundaLetraFix, 
					@cantPrimeraLetraFix, @cantSegundaLetraFix, 
					@posPrimeraLetraFix, @posSegundaLetraFix;
												
	IF @primeraLetraFix IS NULL THEN
		SET fixFigurasPerfectas := 0; 
	END IF;
	
	IF fixFigurasPerfectas > 0 AND @primeraLetraFix IS NOT NULL AND @segundaLetraFix IS NULL THEN
		SET fixFigurasPerfectas := 1; 
	END IF;
	
	-- asigno balota que salieron en las posiciones de la figura y balotas ramdom en las demas
	WHILE @numColums < 26 DO
	
	  IF (@numColums BETWEEN 01 AND 05) THEN 
		  SET @numInit := 01; 
			SET @numEnd := 15;
			SET @letra := 'B';
		  SET @balotas_cols :=  @balotas_b;
	  END IF;
	 
	  IF (@numColums BETWEEN 06 AND 10) THEN 
		  SET @numInit := 16; 
			SET @numEnd := 30;
			SET @letra := 'I';
		  SET @balotas_cols :=  @balotas_i;
	  END IF;
	 
	  IF (@numColums BETWEEN 11 AND 15) THEN 
		  SET @numInit := 31; 
			SET @numEnd := 45;
			SET @letra := 'N';
		  SET @balotas_cols :=  @balotas_n;
	  END IF;
	 
	  IF (@numColums BETWEEN 16 AND 20) THEN 
		  SET @numInit := 46;  
			SET @numEnd := 60;
			SET @letra := 'G';
		  SET @balotas_cols :=  @balotas_g;
	  END IF;
	 
	  IF (@numColums BETWEEN 21 AND 25) THEN 
		  SET @numInit := 61; 
			SET @numEnd := 75;
			SET @letra := 'O';
		  SET @balotas_cols :=  @balotas_o;
	  END IF;

		SET @columna =  CONCAT('P', @numColums);
		SET @balotasSelQuery = SUBSTR(@balotasSeleccionadas, 1, length(@balotasSeleccionadas)-1); -- remove ultimo ,
		
		IF LENGTH(@balotas_cols) < 1 THEN
		   SET @balotas_cols := 99;
	  END IF;
	  
	    SET @balotaSeleccionada = 99;
		
		IF INSTR(@PosicionesColumnas, CONCAT(@columna, ',')) > 0 THEN  -- la posicion del carton esta dentro de las posiciones de la figura 
									
		   SET @numquery := CONCAT( 'SELECT Numero INTO @balotaSeleccionada
			                             FROM TempNumerosCarton 
									                WHERE Numero BETWEEN ', @numInit ,' AND ', @numEnd,'
									                  AND Numero NOT IN(', @balotasSelQuery,') 
									                  AND Numero IN (', @balotas_cols ,')  
									             ORDER BY RAND() LIMIT 1' );
	
		ELSE  -- balotas ramdom -- si figuras fixFigurasPerfectas > 0 incluira solo balotas que hayan salido (columnas con mas balotas)
		
				IF (  fixFigurasPerfectas > 0 
				      AND (  
					          (@letra = @primeraLetraFix AND @numColums = @posPrimeraLetraFix) OR
					          (@letra = @segundaLetraFix AND @numColums = @posSegundaLetraFix) 
							     )
						) THEN 
												
						SET @numquery := CONCAT( 'SELECT Numero INTO @balotaSeleccionada
						                            FROM TempNumerosCarton 
									                     WHERE Numero BETWEEN ', @numInit ,' AND ', @numEnd,'
									                       AND Numero NOT IN(', @balotasSelQuery,') 
															           AND Numero IN (', @balotas_cols ,')   
												            ORDER BY RAND() LIMIT 1' );
														
						SET fixFigurasPerfectas	:= fixFigurasPerfectas - 1;	
						
				ELSE 
											
					  SET @numquery := CONCAT( 'SELECT Numero INTO @balotaSeleccionada
						                            FROM TempNumerosCarton  
									                     WHERE Numero BETWEEN ', @numInit ,' AND ', @numEnd,'
									                       AND Numero NOT IN(', @balotasSelQuery,') 
																         AND Numero NOT IN (', @balotas_cols ,')   
															      ORDER BY RAND() LIMIT 1' );
				
				
				END IF;
											
		END IF;
		
		-- obligar "ultima balota" si esta en la misma columna Y posicion y no ha sido seleeccionada
		IF @UltimaBalotaLetra = @letra AND INSTR(@balotasSeleccionadas, CONCAT( ',',  @UltimaBalota, ',' )) = 0 AND INSTR(@PosicionesColumnas, CONCAT(@columna, ',')) > 0 THEN
		    -- asigno la ultima balota al carton
			SET @balotaSeleccionada := @UltimaBalota;
			
		ELSE
		    -- busco en el query
			PREPARE stmt FROM @numquery;
			EXECUTE stmt;
			DEALLOCATE PREPARE stmt;
			
		END IF;
		
		-- la posicion 13 siempre en 0
		IF @numColums = 13 THEN
		  SET @balotaSeleccionada := 0;
		END IF;
		
		-- FIX PARA LA MAYORIA DE LOSCASOS DE 99
		IF @balotaSeleccionada = '99' THEN
		
				 SET @numquery := CONCAT( 'SELECT Numero INTO @balotaSeleccionada
																		 FROM TempNumerosCarton  
																		WHERE Numero BETWEEN ', @numInit ,' AND ', @numEnd,'
																		  AND Numero NOT IN(', @balotasSelQuery,') 
																			AND Numero IN (', @balotas_cols ,')     
															   ORDER BY RAND() LIMIT 1' );
														
			-- busco en el query
			PREPARE stmt FROM @numquery;
			EXECUTE stmt;
			DEALLOCATE PREPARE stmt;
    END IF;
		
		SET @insColums := CONCAT(@insColums, @balotaSeleccionada, ','); 
		SET @balotasSeleccionadas = CONCAT(@balotasSeleccionadas,  @balotaSeleccionada, ',');
		SET @balotasUpd = CONCAT(@balotasUpd, @columna, '=' , @balotaSeleccionada, ', ');
		SET @numColums := @numColums + 1;
		
		IF @balotaSeleccionada = '99' THEN
			SET @errorBalota := @numquery;
    END IF;	
		
  END WHILE;
	
	-- limpiar seleccionadas para el update
	SET @balotasUpd := SUBSTR(@balotasUpd, 1, length(@balotasUpd)-2); -- remover final ,

  IF INSTR(@balotasUpd, '99') > 0 THEN 
	
		INSERT INTO auditoria(IdUsuario, 
							Tabla, 
							Accion, 
							Complemento,
							IdProgramacionJuego,
							RegistroAnterior,
							RegistroNuevo
							)
					 VALUES(1, 
							'rondasfiguras', 
							'UPD',
							'ERROR PSESPECIAL', 
							pIdJuego,
							CONCAT('balotasUpd: ', @balotasUpd, 
										 ' pIdPlenoAutomatico: ', pIdPlenoAutomatico,
										 ' pCartonAsignado: ', pCartonAsignado,
										 ' errorBalota: ', IFNULL(@errorBalota, '') 
										 ),
							''
							);
			SET salida := '99 ENCONTRADO EN BALOTAS - SALIENDO ';
			LEAVE sp;
	END IF;
	
	-- actualizo la serie para crear el carton ganador
	SET @updCarton := CONCAT ('UPDATE ', pTablaSerie, ' SET ' , @balotasUpd, ' WHERE Numero =', pCartonAsignado); 

	SELECT @updCarton;
	 
	PREPARE stmt FROM @updCarton;
	EXECUTE stmt;
	DEALLOCATE PREPARE stmt;
	 
	-- coloco la figura como no acumulable y no multiple
	 UPDATE rondasfiguras 
	    SET Acumula = 'N', 
			    Multiple = 'N',
			    Carton  = pCartonAsignado
	 WHERE IdFiguraRonda = pIdFiguraRonda;
	 
	 SET salida := concat('ACTUALIZADO/A SERIE/CARTON ', pCartonAsignado, ' ID FIGURA: ', pIdPlenoAutomatico);
	 SET pSalida := salida;
	 SET pCreado := 1;
	 LEAVE sp;	
	 
END
