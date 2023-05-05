USE [DBSSS]
GO

/****** Object:  StoredProcedure [dbo].[INS_SOLICITUDFLUJO_APROBADORES_V2]    Script Date: 5/5/2023 8:26:33 AM ******/
SET ANSI_NULLS ON
GO

SET QUOTED_IDENTIFIER ON
GO

-- =============================================
-- Author:		<Renzo Morante>
-- Create date: <08-06-2020>
-- Description:	<Inserta Aprobadores - Remplazando CP por IDServicio>
-- =============================================
-- exec INS_SOLICITUDFLUJO_APROBADORES_V2 6556,'A-0040839', 2, 0, ''
-- exec INS_SOLICITUDFLUJO_APROBADORES_V2 6556,'001251', 2, 0, ''
-- exec INS_SOLICITUDFLUJO_APROBADORES_V2 6637,'', 2, 0, ''
ALTER PROCEDURE [dbo].[INS_SOLICITUDFLUJO_APROBADORES_V2] @p_IdSolicitud INT
	,@p_OC VARCHAR(20)
	,@p_Estado INT
	,@p_codError INT OUTPUT
	,@p_desError VARCHAR(100) OUTPUT
AS
DECLARE @vCP VARCHAR(10)
	,@VUNIDAD VARCHAR(5)
	,@vIdUsuario INT
	,@vCodUsuario VARCHAR(10)
	,@vDNI VARCHAR(10)
	,@p_IdFlujo INT
	,@cpAntigua INT = 0

BEGIN TRY
	SET @p_IdFlujo = (
			SELECT ISNULL(MAX(IdFlujo), 0) + 1
			FROM dbo.SolicitudFlujo
			)

	IF @p_OC <> ''
		SELECT @cpAntigua = count(*)
		FROM DATA601..C_COMPRA t0
		JOIN DATA601..L_COMPRA t1 ON t0.orden = t1.orden
			AND t0.tip_oc = t1.tip_oc
		WHERE t0.stat = 'A'
			AND t1.STAT = 'A'
			AND t1.TIP_OC + '-' + t1.ORDEN = @p_OC

	PRINT ('valor de @cpAntigua:' + cast(@cpAntigua AS VARCHAR))

	-- Si es una OC antigua de SAPIA
	IF @cpAntigua > 0
	BEGIN
		-- Si es solicitud por OC
		IF @p_OC <> ''
		BEGIN
			SELECT TOP 1 @vCP = t1.NRO_PEDIDO
				,@VUNIDAD = unegocio
			FROM DATA601..C_COMPRA t0
			JOIN DATA601..L_COMPRA t1 ON t0.orden = t1.orden
				AND t0.tip_oc = t1.tip_oc
			WHERE t0.stat = 'A'
				AND t1.STAT = 'A' /*and year(t0.FCH_DOC)='2020'*/
				AND t1.TIP_OC + '-' + t1.ORDEN = @p_OC

			--'I-0008988'
			IF @vCP = '0000001'
			BEGIN
				--Si tiene CP (0000000,0000001) quien generó la solicitud :
				SET @vIdUsuario = (
						SELECT idusuario_creacion
						FROM Contabilidad..LOG_SOLICITUD_OC_CAB
						WHERE oc = @p_OC
							AND bestado = 1
						)
				SET @vCodUsuario = (
						SELECT vUsuario
						FROM sgu..usuario
						WHERE IdUsuario = @vIdUsuario
						)

				EXEC dbo.INS_SOLICITUD_REGISTRA_PERSONAL @vCodUsuario
					,0
					,0
					,''

				IF NOT EXISTS (
						SELECT 1
						FROM dbo.SolicitudFlujo
						WHERE IdSolicitud = @p_IdSolicitud
							AND Flujo = 'Aprobador'
							AND IdUsuario = @vIdUsuario
						)
				BEGIN
					INSERT INTO dbo.SolicitudFlujo (
						IdFlujo
						,IdSolicitud
						,Flujo
						,FechaFlujo
						,IdUsuario
						,IdRol
						,Observacion
						,Estado
						)
					-- obtenemos los datos del usuario
					SELECT DISTINCT @p_IdFlujo
						,@p_IdSolicitud
						,'Aprobador'
						,GETDATE()
						,a.IdUsuario
						,ISNULL(c.IdRol, 165)
						,'Registro de aprobadores'
						,@p_Estado
					FROM sgu..usuario a
					INNER JOIN sgu..rolusuario b ON a.idusuario = b.IdUsuario
					INNER JOIN sgu..rol c ON b.IdRol = c.IdRol
						AND IdSistema = 82
					WHERE a.IdUsuario = @vIdUsuario --810 --107 @vIdUsuario

					--EXEC dbo.INS_SOLICITUD_REGISTRA_PERSONAL @vCodUsuario, 0, 0, ''
					PRINT 'caso 1'
				END

				PRINT ('Usuario encontrado:' + cast(@vIdUsuario AS VARCHAR))
			END
			ELSE IF @vCP = '0000000'
			BEGIN
				INSERT INTO dbo.SolicitudFlujo (
					IdFlujo
					,IdSolicitud
					,Flujo
					,FechaFlujo
					,IdUsuario
					,IdRol
					,Observacion
					,Estado
					)
				SELECT DISTINCT @p_IdFlujo
					,@p_IdSolicitud
					,'Aprobador'
					,GETDATE()
					,a.IdUsuario
					,167
					,
					--c.IdRol,
					'Registro de aprobadores'
					,@p_Estado
				FROM sgu..usuario a
				INNER JOIN sgu..rolusuario b ON a.idusuario = b.IdUsuario
				--inner join sgu..rol c on b.IdRol=c.IdRol and IdSistema=82
				WHERE A.IdUsuario = (
						CASE @VUNIDAD
							WHEN '90304'
								THEN 7004 --122  dgurrieri en vez de eulloa 
							WHEN '90204'
								THEN 936 --7004 --90 --303  --yguillen en vez dgurrieri en vez de jli  
							WHEN '90103'
								THEN 7003 --6990  --1966 
							WHEN '90102'
								THEN 678 -- SE CAMBIAR GGARCIA X amsodani X VAMPUERO
							WHEN '90101'
								THEN 801
							WHEN '90302'
								THEN 2
							WHEN '90201'
								THEN 7003 --6990 --1966
							WHEN '90205'
								THEN 7004 --90 -- dgurrieri en vez de jli
							WHEN '90301'
								THEN 7003 --6990  --1966--103 
							WHEN '90303'
								THEN 678 -- 295 vampuero en vez de mruiz
							WHEN '90202'
								THEN 138
							WHEN '90701'
								THEN 759 --934
							WHEN '90702'
								THEN 759 --934
							WHEN '90704'
								THEN 7003 --CAMBIAR A RUSELL POR MCRIVERA
							WHEN '90705'
								THEN 7018 -- CAMBIAR A MYZIQUE(901) POR FIORELA ROMANI (flquiliche --7018) 
							WHEN '90602'
								THEN 135
							WHEN '80101'
								THEN 7033 -- Steffani quiroz en vez de David 7004 --122 dgurrieri en vez de eulloa
							WHEN '80102'
								THEN 7033 -- Steffani quiroz en vez de David 7004
							WHEN '80103'
								THEN 7033 -- Steffani quiroz en vez de David
							WHEN '80104'
								THEN 7033 -- Steffani quiroz en vez de David 7004
							WHEN '80201'
								THEN 7033 -- Steffani quiroz en vez de David 7004
							WHEN '80202'
								THEN 7033 -- Steffani quiroz en vez de David 7004
							WHEN '90401'
								THEN 2970
							WHEN '90402'
								THEN 2970
							WHEN '90406'
								THEN 1955
							WHEN '90604'
								THEN 932
							WHEN '90412'
								THEN 1960
							WHEN '90606'
								THEN 7036 --CAMBIAR A BETY POR WILBER 899 --887
							WHEN '90407'
								THEN 1955
							WHEN '90413'
								THEN 951
							ELSE 0
							END
						)

				PRINT ('Caso 2  Unidad: ' + @VUNIDAD)
			END
			ELSE IF @vCP <> '0000000'
				OR @vCP <> '0000001'
			BEGIN
				--Si tiene CP(casi siempre empiez con 084...) entonces busca jefe de proyecto
				SET @vDNI = (
						SELECT dni
						FROM Contabilidad..c_pedido_jefe
						WHERE nro_pedido = @vCP /*'0848532'*/
							AND idestado = 1
						)

				-- obtenemos los datos del usuario x dni
				--				SET @vCodUsuario = (select vUsuario from sgu..usuario where vdni = @vDNI)  emmh comentado no se usa en este if
				--EXEC dbo.INS_SOLICITUD_REGISTRA_PERSONAL @vCodUsuario, 0, 0, ''
				IF NOT EXISTS (
						SELECT 1
						FROM dbo.SolicitudFlujo
						WHERE IdSolicitud = @p_IdSolicitud
							AND Flujo = 'Aprobador'
							AND IdUsuario = (
								SELECT a.IdUsuario
								FROM sgu..usuario a
								INNER JOIN sgu..rolusuario b ON a.idusuario = b.IdUsuario
								INNER JOIN sgu..rol c ON b.IdRol = c.IdRol
								WHERE a.vDNI = @vDNI
									AND c.idSistema = 82
								)
						)
				BEGIN
					INSERT INTO dbo.SolicitudFlujo (
						IdFlujo
						,IdSolicitud
						,Flujo
						,FechaFlujo
						,IdUsuario
						,IdRol
						,Observacion
						,Estado
						)
					SELECT DISTINCT @p_IdFlujo
						,@p_IdSolicitud
						,'Aprobador'
						,GETDATE()
						,a.IdUsuario
						,ISNULL(c.IdRol, 165)
						,'Registro de aprobadores'
						,@p_Estado
					/*
								a.vusuario,
								a.vnombre+' '+a.vApePat+' '+a.vApeMat Nombre,
								c.vDescripcion,
								a.nEstado,
								vdni
								*/
					FROM sgu..usuario a
					INNER JOIN sgu..rolusuario b ON a.idusuario = b.IdUsuario
					INNER JOIN sgu..rol c ON b.IdRol = c.IdRol
						AND IdSistema = 82
					WHERE vdni = @vDNI --'42728718'

					--EXEC dbo.INS_SOLICITUD_REGISTRA_PERSONAL @vCodUsuario, 0, 0, ''
					PRINT 'caso 3'
				END

				PRINT 'caso 4 DNI=' + @vDNI
			END

			IF @vIdUsuario = ''
				AND @vDNI = ''
			BEGIN
				--Si no encuentra ninguno de los anteriores por default quien generó la solicitud
				SET @vIdUsuario = (
						SELECT idusuario_creacion
						FROM Contabilidad..LOG_SOLICITUD_OC_CAB
						WHERE oc = @p_OC /*'I-0008988'*/
							AND bestado = 1
						)
				SET @vCodUsuario = (
						SELECT vUsuario
						FROM sgu..usuario
						WHERE IdUsuario = @vIdUsuario
						)

				EXEC dbo.INS_SOLICITUD_REGISTRA_PERSONAL @vCodUsuario
					,0
					,0
					,''

				IF NOT EXISTS (
						SELECT 1
						FROM dbo.SolicitudFlujo
						WHERE IdSolicitud = @p_IdSolicitud
							AND Flujo = 'Aprobador'
							AND IdUsuario = @vIdUsuario
						)
				BEGIN
					INSERT INTO dbo.SolicitudFlujo (
						IdFlujo
						,IdSolicitud
						,Flujo
						,FechaFlujo
						,IdUsuario
						,IdRol
						,Observacion
						,Estado
						)
					SELECT DISTINCT @p_IdFlujo
						,@p_IdSolicitud
						,'Aprobador'
						,GETDATE()
						,a.IdUsuario
						,ISNULL(c.IdRol, 165)
						,'Registro de aprobadores'
						,@p_Estado
					/*
								a.vusuario,
								a.vnombre+' '+a.vApePat+' '+a.vApeMat Nombre,
								c.vDescripcion,
								a.nEstado,
								vdni
								*/
					FROM sgu..usuario a
					INNER JOIN sgu..rolusuario b ON a.idusuario = b.IdUsuario
					INNER JOIN sgu..rol c ON b.IdRol = c.IdRol
						AND IdSistema = 82
					WHERE a.IdUsuario = @vIdUsuario

					--107
					--EXEC dbo.INS_SOLICITUD_REGISTRA_PERSONAL @vCodUsuario, 0, 0, ''
					PRINT 'caso 6'
				END

				PRINT 'caso 7'
			END
		END
		ELSE -- es decir es solicitud por contrato
		BEGIN
			PRINT 'Sin OC'

			DECLARE @idunidad_det VARCHAR(10)

			-- buscamos jefe de proyecto por CP SELECT * FROM Contrato
			SET @vCP = (
					SELECT TOP 1 CP
					FROM dbo.Contrato
					WHERE IdContrato IN (
							SELECT IdContrato
							FROM dbo.SolicitudTipo
							WHERE IdSolicitud = @p_IdSolicitud
								AND IdContrato <> 0
							)
					)
			SET @VUNIDAD = (
					SELECT TOP 1 NroUnidad
					FROM dbo.Contrato
					WHERE IdContrato IN (
							SELECT IdContrato
							FROM dbo.SolicitudTipo
							WHERE IdSolicitud = @p_IdSolicitud
								AND IdContrato <> 0
							)
					)
			SET @vDNI = (
					SELECT dni
					FROM Contabilidad..c_pedido_jefe
					WHERE nro_pedido = @vCP
						AND idestado = 1
					)
			SET @vCodUsuario = (
					SELECT vUsuario
					FROM sgu..usuario a
					INNER JOIN sgu..rolusuario b ON a.idusuario = b.IdUsuario
					INNER JOIN sgu..rol c ON b.IdRol = c.IdRol
					WHERE a.vDNI = @vDNI
						AND c.idSistema = 82
						AND a.nEstado = 1
					)

			--			SET @vCodUsuario = (select vUsuario from sgu..usuario where vdni = @vDNI and nEstado =  1) --comentado por que se repetia usuario julloshi
			IF @vCP <> '0000000'
			BEGIN
				--			IF NOT EXISTS (SELECT 1 FROM dbo.SolicitudFlujo WHERE IdSolicitud = @p_IdSolicitud AND Flujo = 'Aprobador' AND IdUsuario = (select IdUsuario from sgu..usuario where vdni = @vDNI))
				IF NOT EXISTS (
						SELECT 1
						FROM dbo.SolicitudFlujo
						WHERE IdSolicitud = @p_IdSolicitud
							AND Flujo = 'Aprobador'
							AND IdUsuario = (
								SELECT a.IdUsuario
								FROM sgu..usuario a
								INNER JOIN sgu..rolusuario b ON a.idusuario = b.IdUsuario
								INNER JOIN sgu..rol c ON b.IdRol = c.IdRol
								WHERE a.vDNI = @vDNI
									AND c.idSistema = 82
								)
						)
				BEGIN
					IF @vCodUsuario <> ''
					BEGIN
						INSERT INTO dbo.SolicitudFlujo (
							IdFlujo
							,IdSolicitud
							,Flujo
							,FechaFlujo
							,IdUsuario
							,IdRol
							,Observacion
							,Estado
							)
						SELECT DISTINCT @p_IdFlujo
							,@p_IdSolicitud
							,'Aprobador'
							,GETDATE()
							,a.IdUsuario
							,167
							,
							--c.IdRol,
							'Registro de aprobadores'
							,@p_Estado
						FROM sgu..usuario a
						INNER JOIN sgu..rolusuario b ON a.idusuario = b.IdUsuario
						INNER JOIN sgu..rol c ON b.IdRol = c.IdRol
							AND IdSistema = 82
						--descomentado por que se repetia usuario
						WHERE vdni = @vDNI
					END
					ELSE
					BEGIN
						-- buscamos lider de unidad
						WITH t_unidades
						AS (
							SELECT DISTINCT SUBSTRING(NroUnidad, 1, 3) idunidad
							FROM dbo.Contrato
							WHERE IdContrato IN (
									SELECT IdContrato
									FROM dbo.SolicitudTipo
									WHERE IdSolicitud = @p_IdSolicitud
									)
							)
							,t_unidades_p
						AS (
							SELECT ROW_NUMBER() OVER (
									ORDER BY idunidad_det_padre
									) AS item
								,u.idunidad_det_padre
							FROM t_unidades t
							INNER JOIN Contabilidad.DBO.rg_unidad_det u ON t.idunidad = u.unidad
							)
						--select * from Contabilidad.DBO.rg_unidad_det
						SELECT @idunidad_det = idunidad_det_padre
						FROM t_unidades_p

						PRINT @idunidad_det

						INSERT INTO dbo.SolicitudFlujo (
							IdFlujo
							,IdSolicitud
							,Flujo
							,FechaFlujo
							,IdUsuario
							,IdRol
							,Observacion
							,Estado
							)
						SELECT DISTINCT @p_IdFlujo
							,@p_IdSolicitud
							,'Aprobador'
							,GETDATE()
							,IdUsuario
							,168
							,'Registro de aprobadores'
							,@p_Estado
						FROM Contabilidad.DBO.GOP_FUN_FLUJOAPROBACION_X_UNIDAD(@idunidad_det)
						WHERE numerosecuencia = 2;
					END
							--EXEC dbo.INS_SOLICITUD_REGISTRA_PERSONAL @vCodUsuario, 0, 0, ''
				END
			END

			IF @vCP = '0000000'
			BEGIN
				INSERT INTO dbo.SolicitudFlujo (
					IdFlujo
					,IdSolicitud
					,Flujo
					,FechaFlujo
					,IdUsuario
					,IdRol
					,Observacion
					,Estado
					)
				SELECT DISTINCT @p_IdFlujo
					,@p_IdSolicitud
					,'Aprobador'
					,GETDATE()
					,a.IdUsuario
					,167
					,
					--c.IdRol,
					'Registro de aprobadores'
					,@p_Estado
				FROM sgu..usuario a
				INNER JOIN sgu..rolusuario b ON a.idusuario = b.IdUsuario
				--inner join sgu..rol c on b.IdRol=c.IdRol and IdSistema=82
				WHERE A.IdUsuario = (
						CASE @VUNIDAD
							WHEN '90304'
								THEN 7004 --122    dgurrieri en vez de eullo
							WHEN '90204'
								THEN 936 --7004 --90 --303  -- yguillen en vez dgurrieri en vez de jli
							WHEN '90103'
								THEN 7003 --6990  --1966 
							WHEN '90102'
								THEN 678 -- SE CAMBIAR GGARCIA X amsodani x vampuero
							WHEN '90101'
								THEN 801
							WHEN '90302'
								THEN 2
							WHEN '90201'
								THEN 7003 --6990  --1966
							WHEN '90205'
								THEN 7004 --90 -- dgurrieri en vez de jli
							WHEN '90301'
								THEN 103
							WHEN '90303'
								THEN 678 --295 Vampuero en vez de maria ruiz
							WHEN '90202'
								THEN 138
							WHEN '90701'
								THEN 759 --934
							WHEN '90702'
								THEN 759 --934
							WHEN '90704'
								THEN 7003 --CAMBIAR A RUSELL POR MCRIVERA
							WHEN '90705'
								THEN 7018 -- CAMBIAR A MYZIQUE(901) POR FIORELA ROMANI (flquiliche --7018) 
							WHEN '90602'
								THEN 135
							WHEN '80101'
								THEN 7033 --  Steffany Quiroz en vez de David 7004--122 
							WHEN '80102'
								THEN 7033 -- Steffani quiroz en vez de David 7004
							WHEN '80103'
								THEN 7033 -- Steffani quiroz en vez de David 7004
							WHEN '80104'
								THEN 7033 -- Steffani quiroz en vez de David 7004
							WHEN '80201'
								THEN 7033 -- Steffani quiroz en vez de David 7004
							WHEN '80202'
								THEN 7033 -- Steffani quiroz en vez de David 7004
							WHEN '90401'
								THEN 2970
							WHEN '90402'
								THEN 2970
							WHEN '90406'
								THEN 1955
							WHEN '90604'
								THEN 932
							WHEN '90606'
								THEN 7036 --CAMBIAR A BETY POR WILBER 899 --887
							WHEN '90407'
								THEN 1955
							ELSE 0
							END
						)
			END
		END
	END
	ELSE -- Es una OC nueva con ID Servicios y se debe de buscar de la tabla stefanini
	IF @p_OC <> '' -- Si es por OC
	BEGIN
		-- seteamos CP
		SET @vCP = '0000000'

		-- Obtener el Id de Servicio por el nro de la OC
		SELECT TOP 1 @vCP = C7_XPROY
		--from DBOPTAC..SC7010 a
--		FROM [10.161.75.130].dadosper.dbo.SC7010 a
		FROM dadosper.dbo.SC7010 a
		WHERE C7_FILIAL = '0604'
			AND C7_NUM = @p_OC
			AND D_E_L_E_T_ <> '*'

		----Si tiene CP entonces busca jefe de proyecto en la tabla de configuración de valorizaciones
		IF @vCP <> '0000000'
			AND @vCP <> ''
		BEGIN
			PRINT ('CP nueva encontrada:' + @vCP)

			SET @vIdUsuario = 0

			-- Obtener el dni del Usuario de la tabla que tiene los IDServicios de TOTVS
			SELECT @vIdUsuario = isnull(IDUSUARIO_JP, 0)
			FROM Contabilidad..TOTVS_IDSERVICIO
			WHERE idServicio = @vCP

			PRINT ('Usuario encontrado:' + cast(@vIdUsuario AS VARCHAR))
			PRINT ('Caso 11')

			-- Si encontro la CP en esa tabla entonces es un IDServicio de cliente y Obtenemos datos del Usuario SGU
			IF @vIdUsuario > 0
			BEGIN
				--select @vIdUsuario = IdUsuario, @vCodUsuario = vUsuario  from sgu..usuario where vDNI = @vDNI and bestado=1)
				PRINT ('Caso 111')

				IF NOT EXISTS (
						SELECT 1
						FROM dbo.SolicitudFlujo
						WHERE IdSolicitud = @p_IdSolicitud
							AND Flujo = 'Aprobador'
							AND IdUsuario = @vIdUsuario
						)
				BEGIN
					INSERT INTO dbo.SolicitudFlujo (
						IdFlujo
						,IdSolicitud
						,Flujo
						,FechaFlujo
						,IdUsuario
						,IdRol
						,Observacion
						,Estado
						)
					SELECT DISTINCT @p_IdFlujo
						,@p_IdSolicitud
						,'Aprobador'
						,GETDATE()
						,a.IdUsuario
						,ISNULL(c.IdRol, 165)
						,'Registro de aprobadores'
						,@p_Estado
					FROM sgu..usuario a
					INNER JOIN sgu..rolusuario b ON a.idusuario = b.IdUsuario
					INNER JOIN sgu..rol c ON b.IdRol = c.IdRol
						AND IdSistema = 82
					WHERE a.idusuario = @vIdUsuario
				END
			END
			ELSE
			BEGIN
				-- Si NO encontro ASignado un UsuarioJP  IDServicio de que no es cliente, es decir backoffice
				PRINT ('CP nueva no tiene Usuario JP asignado, debe buscar su GUN:' + @vCP)

				SELECT @vIdUsuario = idusuario
				FROM dbo.FUN_FLUJOAPROBACION_X_SERVICIO(@vCP)
				WHERE numerosecuencia = 1

				-- Por ultimo sino encontro buscar en los de gasto
				IF @vIdUsuario = 0
					SELECT @vIdUsuario = CASE 
							-- When @vCP in('010001','010002','102085','107064') Then 0 -- Ariel Pizzo
							--When @vCP in('076174','076176','080698') Then 0  -- Cristina de la Cruz
							WHEN @vCP IN (
									'070896'
									,'070914'
									,'082500'
									)
								THEN 7006 -- Dante Campos
									-- When @vCP in('082501') Then 0  -- Jaime Mourao
							WHEN @vCP IN (
									'070895'
									,'070913'
									)
								THEN 1961 -- Jose Novoa
							WHEN @vCP IN (
									'070900'
									,'076180'
									)
								THEN 7019 -- Juan Alonso Hurtado
							WHEN @vCP IN (
									'010003'
									,'010004'
									,'099646'
									,'099655'
									)
								THEN 2970 -- luis Torres
							WHEN @vCP IN (
									'076172'
									,'080702'
									,'081955'
									,'087318'
									,'087327'
									,'087328'
									)
								THEN 7003 -- Maria Cristina Rivera
							WHEN @vCP IN (
									'010005'
									,'010006'
									,'099659'
									,'099675'
									)
								THEN 1961 --Jose Novoa antes  -- Mario Hidalgo antes
									--When @vCP in('070898','070916') Then 0  -- Paola Mora
									--When @vCP in('076181','080703','100794') Then 0  -- Raul Echevarria
							WHEN @vCP IN (
									'069564'
									,'070903'
									,'070915'
									,'100497'
									,'100498'
									)
								THEN 7033 -- Steffany Quiroz
							WHEN @vCP IN (
									'076173'
									,'099642'
									)
								THEN 7036 -- Wilber Cahuachi
							ELSE 2 -- Debe llegar a Eduardo Ramirez
							END

				-- 
				PRINT ('Usuario encontrado:' + cast(@vIdUsuario AS VARCHAR))
				PRINT ('Caso 12')

				INSERT INTO dbo.SolicitudFlujo (
					IdFlujo
					,IdSolicitud
					,Flujo
					,FechaFlujo
					,IdUsuario
					,IdRol
					,Observacion
					,Estado
					)
				SELECT DISTINCT @p_IdFlujo
					,@p_IdSolicitud
					,'Aprobador'
					,GETDATE()
					,a.IdUsuario
					,167
					,
					--c.IdRol,
					'Registro de aprobadores'
					,@p_Estado
				FROM sgu..usuario a
				INNER JOIN sgu..rolusuario b ON a.idusuario = b.IdUsuario
				--inner join sgu..rol c on b.IdRol=c.IdRol and IdSistema=82
				WHERE A.IdUsuario = @vIdUsuario
			END
		END
		ELSE
			-- Si no encontro CP entonces son CPs de Gasto, es decir NO son CPs de clientes
		BEGIN
			SELECT @vIdUsuario = CASE 
					-- When @vCP in('010001','010002','102085','107064') Then 0 -- Ariel Pizzo
					--When @vCP in('076174','076176','080698') Then 0  -- Cristina de la Cruz
					WHEN @vCP IN (
							'070896'
							,'070914'
							,'082500'
							)
						THEN 7006 -- Dante Campos
							-- When @vCP in('082501') Then 0  -- Jaime Mourao
					WHEN @vCP IN (
							'070895'
							,'070913'
							)
						THEN 1961 -- Jose Novoa
					WHEN @vCP IN (
							'070900'
							,'076180'
							)
						THEN 7019 -- Juan Alonso Hurtado
					WHEN @vCP IN (
							'010003'
							,'010004'
							,'099646'
							,'099655'
							)
						THEN 2970 -- luis Torres
					WHEN @vCP IN (
							'076172'
							,'080702'
							,'081955'
							,'087318'
							,'087327'
							,'087328'
							)
						THEN 7003 -- Maria Cristina Rivera
					WHEN @vCP IN (
							'010005'
							,'010006'
							,'099659'
							,'099675'
							)
						THEN 1961 --Jose novoa antes era 889  -- Mario Hidalgo
							--When @vCP in('070898','070916') Then 0  -- Paola Mora
							--When @vCP in('076181','080703','100794') Then 0  -- Raul Echevarria
					WHEN @vCP IN (
							'069564'
							,'070903'
							,'070915'
							,'100497'
							,'100498'
							)
						THEN 7033 -- Steffany Quiroz
					WHEN @vCP IN (
							'076173'
							,'099642'
							)
						THEN 7036 -- Wilber Cahuachi
					ELSE 2 -- Debe llegar a Eduardo Ramirez
					END

			-- 
			PRINT ('Usuario encontrado:' + cast(@vIdUsuario AS VARCHAR))
			PRINT ('Caso 13')

			IF NOT EXISTS (
					SELECT 1
					FROM dbo.SolicitudFlujo
					WHERE IdSolicitud = @p_IdSolicitud
						AND Flujo = 'Aprobador'
						AND IdUsuario = @vIdUsuario
					)
			BEGIN
				INSERT INTO dbo.SolicitudFlujo (
					IdFlujo
					,IdSolicitud
					,Flujo
					,FechaFlujo
					,IdUsuario
					,IdRol
					,Observacion
					,Estado
					)
				SELECT DISTINCT @p_IdFlujo
					,@p_IdSolicitud
					,'Aprobador'
					,GETDATE()
					,a.IdUsuario
					,167
					,
					--c.IdRol,
					'Registro de aprobadores'
					,@p_Estado
				FROM sgu..usuario a
				INNER JOIN sgu..rolusuario b ON a.idusuario = b.IdUsuario
				--inner join sgu..rol c on b.IdRol=c.IdRol and IdSistema=82
				WHERE A.IdUsuario = @vIdUsuario
			END
		END
	END
	ELSE -- Si es por Contrato
	BEGIN
		-- seteamos CP
		SET @vCP = '0000000'
		-- Obtenemos la CP de la tabla Contrato por el IDSolicitud
		SET @vCP = (
				SELECT TOP 1 CP
				FROM dbo.Contrato
				WHERE IdContrato IN (
						SELECT IdContrato
						FROM dbo.SolicitudTipo
						WHERE IdSolicitud = @p_IdSolicitud
							AND IdContrato <> 0
						)
				)

		----Si tiene CP entonces busca jefe de proyecto en la tabla de configuración de valorizaciones
		IF @vCP <> '0000000'
			AND @vCP <> ''
		BEGIN
			SET @vIdUsuario = 0

			-- Obtener el dni del Usuario de la tabla que tiene los IDServicios de TOTVS
			SELECT @vIdUsuario = IDUSUARIO_JP
			FROM Contabilidad..TOTVS_IDSERVICIO
			WHERE idServicio = @vCP

			PRINT ('CP nueva encontrada:' + @vCP)
			PRINT ('Usuario encontrado:' + cast(@vIdUsuario AS VARCHAR))
			PRINT ('Caso 14')

			-- Si encontro la CP en esa tabña entonces es un IDServicio de cliente y Obtenemos datos del Usuario SGU
			IF @vIdUsuario > 0
				--select @vIdUsuario = IdUsuario, @vCodUsuario = vUsuario  from sgu..usuario where vDNI = @vDNI and bestado=1)
				IF NOT EXISTS (
						SELECT 1
						FROM dbo.SolicitudFlujo
						WHERE IdSolicitud = @p_IdSolicitud
							AND Flujo = 'Aprobador'
							AND IdUsuario = @vIdUsuario
						)
				BEGIN
					INSERT INTO dbo.SolicitudFlujo (
						IdFlujo
						,IdSolicitud
						,Flujo
						,FechaFlujo
						,IdUsuario
						,IdRol
						,Observacion
						,Estado
						)
					SELECT DISTINCT @p_IdFlujo
						,@p_IdSolicitud
						,'Aprobador'
						,GETDATE()
						,a.IdUsuario
						,ISNULL(c.IdRol, 165)
						,'Registro de aprobadores'
						,@p_Estado
					/*
										a.vusuario,
										a.vnombre+' '+a.vApePat+' '+a.vApeMat Nombre,
										c.vDescripcion,
										a.nEstado,
										vdni
										*/
					FROM sgu..usuario a
					INNER JOIN sgu..rolusuario b ON a.idusuario = b.IdUsuario
					INNER JOIN sgu..rol c ON b.IdRol = c.IdRol
						AND IdSistema = 82
					WHERE a.idusuario = @vIdUsuario
				END
				ELSE
				BEGIN
					-- Si NO encontro la CP en esa tabla entonces es un IDServicio de que no es cliente, es decir backoffice
					SELECT @vIdUsuario = CASE 
							-- When @vCP in('010001','010002','102085','107064') Then 0 -- Ariel Pizzo
							--When @vCP in('076174','076176','080698') Then 0  -- Cristina de la Cruz
							WHEN @vCP IN (
									'070896'
									,'070914'
									,'082500'
									)
								THEN 7006 -- Dante Campos
									-- When @vCP in('082501') Then 0  -- Jaime Mourao
							WHEN @vCP IN (
									'070895'
									,'070913'
									)
								THEN 1961 -- Jose Novoa
							WHEN @vCP IN (
									'070900'
									,'076180'
									)
								THEN 7019 -- Juan Alonso Hurtado
							WHEN @vCP IN (
									'010003'
									,'010004'
									,'099646'
									,'099655'
									)
								THEN 2970 -- luis Torres
							WHEN @vCP IN (
									'076172'
									,'080702'
									,'081955'
									,'087318'
									,'087327'
									,'087328'
									)
								THEN 7003 -- Maria Cristina Rivera
							WHEN @vCP IN (
									'010005'
									,'010006'
									,'099659'
									,'099675'
									)
								THEN 1961 --Jose novoa antes era 889  -- Mario Hidalgo
									--When @vCP in('070898','070916') Then 0  -- Paola Mora
									--When @vCP in('076181','080703','100794') Then 0  -- Raul Echevarria
							WHEN @vCP IN (
									'069564'
									,'070903'
									,'070915'
									,'100497'
									,'100498'
									)
								THEN 7033 -- Steffany Quiroz
							WHEN @vCP IN (
									'076173'
									,'099642'
									)
								THEN 7036 -- Wilber Cahuachi
							ELSE 2 -- Debe llegar a Eduardo Ramirez
							END

					-- 
					PRINT ('Usuario no encontrado, se asigno el de Eduardo:' + cast(@vIdUsuario AS VARCHAR))
					PRINT ('Caso 15')

					INSERT INTO dbo.SolicitudFlujo (
						IdFlujo
						,IdSolicitud
						,Flujo
						,FechaFlujo
						,IdUsuario
						,IdRol
						,Observacion
						,Estado
						)
					SELECT DISTINCT @p_IdFlujo
						,@p_IdSolicitud
						,'Aprobador'
						,GETDATE()
						,a.IdUsuario
						,167
						,
						--c.IdRol,
						'Registro de aprobadores'
						,@p_Estado
					FROM sgu..usuario a
					INNER JOIN sgu..rolusuario b ON a.idusuario = b.IdUsuario
					--inner join sgu..rol c on b.IdRol=c.IdRol and IdSistema=82
					WHERE A.IdUsuario = @vIdUsuario
				END
		END
		ELSE
		BEGIN
			-- SI la CP es '0000000' obtenemos el idresponsable del contrato
			-- Obtenemos el responsable de la tabla Contrato por el IDSolicitud
			SET @vIdUsuario = 0
			SET @vIdUsuario = (
					SELECT TOP 1 idResponsable
					FROM dbo.Contrato
					WHERE IdContrato IN (
							SELECT IdContrato
							FROM dbo.SolicitudTipo
							WHERE IdSolicitud = @p_IdSolicitud
								AND IdContrato <> 0
							)
					)

			PRINT ('Usuario responsable encontrado:' + cast(@vIdUsuario AS VARCHAR))
			PRINT ('Caso 115')

			IF @vIdUsuario = 0
			BEGIN
				PRINT ('116')

				-- Si no encontro usuario responsable del contrato, entonces se lo asigno a Eduardo
				SET @vIdUsuario = 2

				PRINT ('Usuario NO encontrado:' + cast(@vIdUsuario AS VARCHAR))
				PRINT ('Caso 15')
			END

			IF NOT EXISTS (
					SELECT 1
					FROM dbo.SolicitudFlujo
					WHERE IdSolicitud = @p_IdSolicitud
						AND Flujo = 'Aprobador'
						AND IdUsuario = @vIdUsuario
					)
			BEGIN
				INSERT INTO dbo.SolicitudFlujo (
					IdFlujo
					,IdSolicitud
					,Flujo
					,FechaFlujo
					,IdUsuario
					,IdRol
					,Observacion
					,Estado
					)
				SELECT DISTINCT @p_IdFlujo
					,@p_IdSolicitud
					,'Aprobador'
					,GETDATE()
					,a.IdUsuario
					,167
					,
					--c.IdRol,
					'Registro de aprobadores'
					,@p_Estado
				FROM sgu..usuario a
				INNER JOIN sgu..rolusuario b ON a.idusuario = b.IdUsuario
				--inner join sgu..rol c on b.IdRol=c.IdRol and IdSistema=82
				WHERE A.IdUsuario = @vIdUsuario
			END
					--End	
		END
	END

	SET @p_codError = 0
	SET @p_desError = 'El aprobador se registro correctamente'
END TRY

BEGIN CATCH
	SELECT @p_codError = ERROR_NUMBER()
		,@p_desError = ERROR_MESSAGE()
END CATCH
