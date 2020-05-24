# StoredProcs

Arquivos Texto com a stored procedure e a chama da do progama C#

USE [Hrz]
GO
/****** Object:  StoredProcedure [dbo].[spAssetTypes_Save]    Script Date: 5/23/2020 11:01:42 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
ALTER PROCEDURE [dbo].[spAssetTypes_Save]  	
	--	Parameters
	@Id				int           null,
	@Description	varchar( 24)  null,
	-- Row unique identifier
	@RowId			int = 0,
	-- Return code and message
	@return_code     int output,
	@return_identity int output,
	@return_message  varchar( 100) output
as
BEGIN
	-- SET NOCOUNT ON added to prevent extra result sets 
	SET NOCOUNT ON;
	-- Prevents error 	
	set @RowId = ISNULL( @RowId, 0);
	set @Description = ISNULL(NULLIF( @Description,' '), NULL)

	-- Checks for duplicate keys
	if ( @RowId = 0 ) -- INCLUIR este registro
		begin try
			INSERT into AssetTypes 
				( Id, Description ) values ( @Id, @Description );
			--
			set @return_identity = @@Identity;
			set @return_code     = ISNULL( ERROR_NUMBER(), 0);
			set @return_message  = 'Registro ID :' + LTRIM(Str( @@Identity)) + ' incluido OK';
			Return  1;				
		end try
		begin catch
			set @return_identity = @@Identity;			
			set @return_code     = ISNULL( ERROR_NUMBER(), -1);			
			set @return_message  = LTRIM(STR ( @RowId, 5)) + ' : ' + ERROR_MESSAGE() ;
			Return -1;			
		end catch;
	else             -- Modificar o registro com a RowId informada
		begin try
			if Exists ( Select RowId from AssetTypes where RowId = @RowId)			
				begin try 				
					UPDATE AssetTypes 
					   set Id          = ISNULL( @Id,          Id          ),
						   Description = ISNULL( @Description, Description )
					 where RowId = @RowId;
					--
					set @return_identity = ISNULL( @@Identity, @RowId);
					set @return_code     = 0;
					set @return_message  = 'Registro ID : ' + LTRIM(Str( @RowId)) + ' atualizado OK';					
					Return  2;			
				end try 
				begin catch
					set @return_identity = @@Identity;
					set @return_code     = ISNULL( ERROR_NUMBER(), -2);
					set @return_message  = Ltrim(Str( @RowId, 5)) + ' ' + @Description + ' : ' + ERROR_MESSAGE();
					Return -2;
				end catch ;	
			else	--	Nao achou a RowId do registro par atualizar
				begin
					set @return_identity = ISNULL( @@Identity, @RowId);
					set @return_code     = ISNULL( ERROR_NUMBER(), -3);
					set @return_message  = Ltrim(Str( @RowId, 5)) + ' : nao encontrou o registro para atualizar';
					Return -3;				
				end
		end try		
		begin catch
			set @return_identity = @@Identity;
			set @return_code     = ERROR_NUMBER();
			set @return_message  = Ltrim(Str( @RowId, 5)) + ' ' + @Description + ' : ' + ERROR_MESSAGE();
			Return -4;
		end catch; 
END


Chamada do programa C#


 
using System;
using System.Collections.Generic;
using System.Data;
using System.Data.SqlClient;
´´´
´´´     Retorna das msgs da Procedures
        public class CodeMsg
        {
            public int    RetValue { get; set; }
            public string RowId    { get; set; }
            public string ErrCode  { get; set; }
            public string ErrMsg   { get; set; }

            public CodeMsg()
            {
                RetValue = -99;
                RowId    = "-1";
                ErrCode  = "-99";
                ErrMsg   = "Erro na execucao do comando";
            }
        }


´´´       Procedimento de chamada de Stored Procedure

           public CodeMsg AssetType_Save( AssetType c)
            {
                CodeMsg cm = new CodeMsg();
                try
                {
                    using (SqlConnection cn = new SqlConnection(CnnString))
                    using (SqlCommand cmd = new SqlCommand("spAssetTypes_Save", cn))
                    {
                        cmd.CommandType = System.Data.CommandType.StoredProcedure;
                        cmd.Parameters.Clear();

                        cmd.Parameters.AddWithValue("@Id", c.Id);
                        cmd.Parameters.AddWithValue("@Description", c.Description);
                        cmd.Parameters.AddWithValue("@RowId", c.RowId);

                        SqlParameter rowGuid = new SqlParameter("@return_identity", SqlDbType.Int)
                        {
                            Direction = ParameterDirection.Output
                        };
                        cmd.Parameters.Add(rowGuid);
                        SqlParameter errCode = new SqlParameter("@return_code", SqlDbType.Int)
                        {
                            Direction = ParameterDirection.Output
                        };
                        cmd.Parameters.Add(errCode);
                        SqlParameter errMsg = new SqlParameter("@return_message", SqlDbType.NVarChar, 100)
                        {
                            Direction = ParameterDirection.Output
                        };
                        cmd.Parameters.Add(errMsg);

                        cn.Open();
                        cm.RetValue = cmd.ExecuteNonQuery();
                        cm.ErrMsg = errMsg.Value.ToString();
                        cm.ErrCode = errCode.Value.ToString();
                        cm.RowId = rowGuid.Value.ToString();
                    }   
                }
                
                catch (Exception ex)
                {
                    throw ex;
                }
                return cm;
            }
